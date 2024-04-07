# Mental Poker Part 6: Shuffling Implementation

For an overview on Mental Poker, see [Mental Poker Part 0: An Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html).
Other articles in this series: <https://vladris.com/writings/index.html#mental-poker>.
In the previous [post in the series](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html)
we covered the state machine we use to implement game logic.

We now have all the pieces in place to look at a card shuffling algorithm.
Shuffling cards in a game of Mental Poker is one of the key innovations for
this type of zero-trust games. We went over the cryptography aspects of
shuffling in [Part 1](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html).

Let's review the algorithm:

> * Alice takes a deck of cards (an array), shuffles the deck, generates
    a secret key $K_A$, and encrypts each card with $K_A$.
> * Alice hands the shuffled and encrypted deck to Bob. At this point,
    Bob doesn't know what order the cards are in (since Alice encrypted
    the cards in the shuffled deck).
> * Bob takes the deck, shuffles it, generates a secret key $K_B$, and
    encrypts each card with $K_B$.
> * Bob hands the deck to Alice. At this point, neither Alice nor Bob
    know what order the cards are in. Alice got the deck back reshuffled
    and re-encrypted by Bob, so she no longer knows where each card
    ended up. Bob reshuffled an encrypted deck, so he also doesn't know
    where each card is.
>
> At this point the cards are shuffled. In order to play, Alice and Bob
  also need the capability to look at individual cards. In order to enable
  this, the following steps must happen:
>
> * Alice decrypts the shuffled deck with her secret key $K_A$. At this
    point she still doesn't know where each card is, as cards are still
    encrypted with $K_B$.
> * Alice generates a new set of secret keys, one for each card in the
    deck. Assuming a 52-card deck, she generates
    $K_{A_1} ... K_{A_{52}}$ and encrypts each card in the deck with one
    of the keys.
> * Alice hands the deck of cards to Bob. At this point, each card is
    encrypted by Bob's key, $B_K$, and one of Alice's keys, $K_{A_i}$.
> * Bob decrypts the cards using his key $K_B$. He still doesn't know
    where each card is, as now the cards are encrypted with Alice's
    keys.
> * Bob generates another set of secret keys, $K_{B_1} ... K_{B_{52}}$,
    and encrypts each card in the deck.
> * Now each card in the deck is encrypted with a unique key that only
    Alice knows and a unique key only Bob knows.
>
> If Alice wants to look at a card, she asks Bob for his key for that
  card. For example, if Alice draws the first card, encrypted with
  $K_{A_1}$ and $K_{B_1}$, she asks Bob for $K_{B_1}$. If Bob sends her
  $K_{B_1}$, she now has both keys to decrypt the card and "look" at it.
  Bob still can't decrypt it because he doesn't have $K_{A_1}$.
>
> This way, as long as both Alice and Bob agree that one of them is
  supposed to "see" a card, they exchange keys as needed to enable this.

## Implementation

While we covered the algorithm before, we didn't have the infrastructure in
place to implement this. We now do.

### Types

We'll start by describing our shuffle actions. As we just saw in the above
recap, we have 2 steps:

```typescript
type ShuffleAction1 = BaseAction & { type: "Shuffle1"; deck: string[] };
type ShuffleAction2 = BaseAction & { type: "Shuffle2"; deck: string[] };
```

We only need to pass around the deck of cards (encrypted or not), so we extend
the `BaseAction` type (which includes `ClientId` and `type`) to pin the type
and add the deck.

We need more data in the context though:

```typescript
type ShuffleContext = {
    clientId: string;
    deck: string[];
    imFirst: boolean;
    keyProvider: KeyProvider;
    commonKey?: SRAKeyPair;
    privateKeys?: SRAKeyPair[];
};
```

We need to know our `clientId`, whether we are first or second in the turn
order, we need a `keyProvider` to generate encryption keys, a `commonKey`
(that's for the first encryption step) and `privateKeys` (for the second
encryption step). We'll use the context later on, when we stich everything
together. Before that, let's look at the basic shuffling functions.

### Shuffling primitives

First, we need a function that shuffles an array:

```typescript
function shuffleArray<T>(arr: T[]): T[] {
    let currentIndex = arr.length,  randomIndex;
  
    while (currentIndex > 0) {
  
      randomIndex = Math.floor(Math.random() * currentIndex);
      currentIndex--;
  
      [arr[currentIndex], arr[randomIndex]] = [arr[randomIndex], arr[currentIndex]];
    }
  
    return arr;
};
```

We won't go into the details of this, as it's a generic shuffling function, not
specific to Mental Poker, but a required piece.

Let's look at the two shuffling steps next. First step, in which we shuffle
and encrypt all cards with the same key:

```typescript
async function shuffle1(keyProvider: KeyProvider, deck: string[]): Promise<[SRAKeyPair, string[]]> {
    const commonKey = keyProvider.make();

    deck = shuffleArray(deck.map((card) => SRA.encryptString(card, commonKey)));

    return [commonKey, deck];
};
```

The `shuffle1()` function takes a `keyProvider`, a `deck`, and returns a
promise of a shuffled deck plus the key used to encrypt it.

The function is pretty straight-forward: we generate a new key, we encrypt
each card with it, then we shuffle the deck. We return the key and the now
shuffled and encrypted deck.

Both players need to perform the first step, after which both Alice and Bob
have encrypted the deck with $K_A$ and $K_B$ respectively, so neither
knows the order of the cards.

The next step, according to our algorithm, is for each player to decrypt the
deck with their key and encrypt each card individually with a unique key:

```typescript
async function shuffle2(commonKey: SRAKeyPair, keyProvider: KeyProvider, deck: string[]): Promise<[SRAKeyPair[], string[]]> {
    const privateKeys: SRAKeyPair[] = [];

    deck = deck.map((card) => SRA.decryptString(card, commonKey));

    for (let i = 0; i < deck.length; i++) {
        privateKeys.push(keyProvider.make());
        deck[i] = SRA.encryptString(deck[i], privateKeys[i]);
    }

    return [privateKeys, deck];
}
```

`shuffle2()` is also fairly straight-forward. It takes the `commonKey` from
step 1, a `keyProvider`, and the encrypted `deck`.

First, it decrypts all cards using the `commonKey` (note the cards are still
encrypted by the other player). Next, it uses the `keyProvider` to generate
a key for each card, and encrypts each card with the key. The function returns
the private keys generated, and the re-encrypted deck.

We now have all the basics in place. Here's how we put it all together:

### Shuffling state machine

Here is the state machine that describes the shuffling steps:

```typescript
function makeShuffleSequence() {
    return sm.sequence([
        sm.local(async (queue: IQueue<ShuffleAction1>, context: ShuffleContext) => {
            if (!context.imFirst) {
                return;
            }

            [context.commonKey, context.deck] = await shuffle1(context.keyProvider, context.deck);

            await queue.enqueue({
                type: "Shuffle1",
                clientId: context.clientId,
                deck: context.deck,
            });
        }),
        sm.transition(async (action: ShuffleAction1, context: ShuffleContext) => {
            if (action.type !== "Shuffle1") {
                throw new Error("Invalid action type");
            }

            context.deck = action.deck;
        }),
        sm.local(async (queue: IQueue<ShuffleAction1>, context: ShuffleContext) => {
            if (context.imFirst) {
                return;
            }

            [context.commonKey, context.deck] = await shuffle1(context.keyProvider, context.deck);

            await queue.enqueue({
                type: "Shuffle1",
                clientId: context.clientId,
                deck: context.deck,
            });
        }),
        sm.transition(async (action: ShuffleAction1, context: ShuffleContext) => {
            if (action.type !== "Shuffle1") {
                throw new Error("Invalid action type");
            }

            context.deck = action.deck;
        }),
        sm.local(async (queue: IQueue<ShuffleAction2>, context: ShuffleContext) => {
            if (!context.imFirst) {
                return;
            }

            [context.privateKeys, context.deck] = await shuffle2(context.commonKey!, context.keyProvider, context.deck);

            await queue.enqueue({
                type: "Shuffle2",
                clientId: context.clientId,
                deck: context.deck,
            });
        }),
        sm.transition(async (action: ShuffleAction2, context: ShuffleContext) => {
            if (action.type !== "Shuffle2") {
                throw new Error("Invalid action type");
            }

            context.deck = action.deck;
        }),
        sm.local(async (queue: IQueue<ShuffleAction2>, context: ShuffleContext) => {
            if (context.imFirst) {
                return;
            }

            [context.privateKeys, context.deck] = await shuffle2(context.commonKey!, context.keyProvider, context.deck);

            await queue.enqueue({
                type: "Shuffle2",
                clientId: context.clientId,
                deck: context.deck,
            });
        }),
        sm.transition(async (action: ShuffleAction2, context: ShuffleContext) => {
            if (action.type !== "Shuffle2") {
                throw new Error("Invalid action type");
            }

            context.deck = action.deck;
        })
    ]);
}
```

Note we are limiting this to a 2-player game, though we can easily generalize
to more players if needed.

This is a longer function so let's break it down:

* We start with a local transition: if we are not the first player (based on
  some previously established turn order), we do nothing. Else we run
  `shuffle1()` and post the encrypted deck as a `Shuffle1` action.
* Next, we expect a `Shuffle1` action to arrive - either the one we just posted
  (if `imFirst` is `true`) or incoming from the other player. We store the
  encrypted and shuffled deck.
* Then, we call `shuffle1()` if we are *not* the first player - if we are *not*
  the first player, then it is our turn to shuffle now. We post another
  `Shuffle1` action.
* We again expect a `Shuffle1` action to arrive and update the deck.

At this point, both players performed the first step of the shuffle, so the
deck is encrypted with $K_A$ And $K_B$ and neither players knows the turn order.
We move on to the second step of the shuffle, where each player calls
`shuffle2()` to decrypt the deck and re-encrypt each individual card. Again,
depending on whether we are first or not, we take action or wait:

* If `imFirst` is `true`, call `shuffle2()` and post a `Shuffle2` action.
* Expect a `Shuffle2` action and update the deck.
* If `imFirst` is *not* `true`, call `shuffle2()` and post a `Shuffle2` action.
* Expect a `Shuffle2` action and update the deck.

A helper function to run this state machine given an async queue:

```typescript
async function shuffle(
    clientId: string,
    turnOrder: string[],
    sharedPrime: bigint,
    deck: string[],
    actionQueue: IQueue<BaseAction>,
    keySize: number = 128 // Key size, defaults to 128 bytes
): Promise<[SRAKeyPair[], string[]]> {
    if (turnOrder.length !== 2) {
        throw new Error("Shuffle only implemented for exactly two players");
    }

    const context: ShuffleContext = { 
        clientId, 
        deck, 
        imFirst: clientId === turnOrder[0],
        keyProvider: new KeyProvider(sharedPrime, keySize)
    };
    
    const shuffleSequence = makeShuffleSequence();

    await sm.run(shuffleSequence, actionQueue, context);

    return [context.privateKeys!, context.deck];
}
```

We need our `clientId`, the turn order (whether we go first or not), a shared
large prime (to seed other encryption keys), an unshuffled deck, a queue, and,
optionally, a `keySize`.

From the input, we create a `ShuffleContext` with the required data, then we
generate the state machine by calling the function we discussed previously,
and we run the state machine using the given `actionQueue` and generated
`context`.

We return the private keys with which we encrypted each individual card, and
the shuffled and encrypted deck.

### Notes on performance

Shuffling a full deck of 52 cards with large enough key sizes gets noticeably
slow. Note that we need to generate an encryption key for each card, which
involves searching for large prime numbers. The more secure we want the
encryption to be, the larger the number of bits we want in the key, the longer
it takes to find a key.

This can be mitigated with some loading/progress UI while shuffling. For the
[demo discard game](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/discard)
in `mental-poker-toolkit`, I used a smaller deck (only cards from `9` to `A`)
and a smaller key size (64 bits).

When implementing a game, it might be a good idea to start generating
encryption keys asynchronously as soon as possible - note though that the
players need to agree on a shared large prime before key generation can begin.

## Summary

In this post we looked at an implementation of card shuffling.

* We recapped the shuffling algorithm for Mental Poker, which enables playing
  zero-trust card games.
* We implemented the two steps of shuffling as `shuffle1()` and `shuffle2()`.
* We defined the state machine that models 2-player shuffling.
* We went over a helper function that runs the state machine and outputs the
  shuffled deck.
* We briefly discussed performance of shuffling.

The Mental Poker Toolkit is [here](https://github.com/vladris/mental-poker-toolkit).
This post covered card shuffling, which is implemented in the `primitives`
package in [shuffle.ts](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/primitives/src/shuffle.ts).
