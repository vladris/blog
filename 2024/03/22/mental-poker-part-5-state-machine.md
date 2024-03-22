# Mental Poker Part 5: State Machine

For an overview on Mental Poker, see [Mental Poker Part 0: An Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html).
Other articles in this series: <https://vladris.com/writings/index.html#mental-poker>.
In the previous [post in the series](https://vladris.com/blog/2024/03/16/mental-poker-part-4-actions-and-async-queue.html)
we covered *actions* and an async queue implementation.

In this post, we'll finally look at the infrastructure on top of which we'll
model games. The type of games we're considering can all be modeled as state
machines[^1]. The challenge is we need a generic enough framework that works
for any game, so let's consider what they all have in common.

## Transitions

We can't tell what the exact states of a game are, as they depend on the
specific game. But, in general, game play implies transitioning from one
state to another.

### Local transitions

In some cases, an action originates on our client. For example: we pick between
rock, paper, or scissors; we want to draw a card etc. This means we need to run
some logic on our client, then send an Action over our transport to other
clients.

To keep things generic and unopinionated, the minimal interface for this is a
function that takes an action queue and a `context`.

```typescript
type LocalTransition<TAction extends BaseAction, TContext> = (
    actionQueue: IQueue<TAction>,
    context: TContext
) => void | Promise<void>;
```

We covered the queue in the [previous post](https://vladris.com/blog/2024/03/16/mental-poker-part-4-actions-and-async-queue.html).
We need this in a local transition because we will run some code then, in most
cases, we'll want to enqueue an action and send it to other players. We'll look
at an example of this later on in this post.

The `context` can be anything - this enables the game to pass-through whatever
data the function needs. Our state machine implementation doesn't care about
what that data is, this is just the mechanism to make it available to the code
in the function.

The function can return either `void` or a `Promise<void>` in case it needs to
be async.

### Remote transitions

In other cases, an action arrives over the transport. This is an action that
was sent either by another player, or by us and we receive it back from the
server after it has been sequenced[^2].

In this case, our interface is a function that takes the incoming `Action` and
a context.

```typescript
type Transition<TAction extends BaseAction, TContext> = (
    action: TAction,
    context: TContext
) => void | Promise<void>;
```

In this case, we don't necessarily need access to the queue, since we won't
enqueue an action, rather we're processing one. The `context` is, again, up to
the consumer of this API.

The function similarly returns `void` or a `Promise<void>` in case it needs to
be async.

### Runnable transition

Finally, we need an abstraction over both `LocalTransition` and `Transition` so
when we specify our state machine we can treat them the same way. We'll use
`RunnableTransition` for this:

```typescript
type RunnableTransition<TContext> = {
    actionQueue: IQueue<BaseAction>,
    context: TContext
}: Promise<void>;
```

We expect users of our library to write code in terms of local transitions
(`LocalTransition`) and remote transitions (`Transition`). This type is meant
to be used internally. Note we are doing some type erasure here as we're going
from a generic `IQueue` to a `IQueue<BaseAction>`. That's because we need to
work with the queue in our library code, but the exact `Action` types depend on
the game.

For local transitions, we simply pass through the `actionQueue`. For remote
transitions, we dequeue an action and pass that. We'll see how to do this next.

We're also normalizing return to be `Promise<void>` regardless of whether the
transition function originally returned `void` or `Promise<void>`.

## State Machine

Our state machine is implemented as a set of functions. First, we have a few
factory functions. `local()` creates a `RunnableTransition` from a
`LocalTransition`:

```typescript
function local<TAction extends BaseAction, TContext>(
    transition: LocalTransition<TAction, TContext>
): RunnableTransition<TContext> {
    return async (queue: IQueue<BaseAction>, context: TContext) =>
        await Promise.resolve(
            transition(queue as IQueue<TAction>, context)
        );
}
```

We call `Promise.resolve()` to get a `Promise` regardless of whether the given
`transition` is a synchronous or asynchronous function.

`remote()` converts a remote transition into a `RunnableTransition`:

```typescript
function transition<TAction extends BaseAction, TContext>(
    transition: Transition<TAction, TContext>
): RunnableTransition<TContext> {
    return async (queue: IQueue<BaseAction>, context: TContext) => {
        const action = await queue.dequeue();
        await Promise.resolve(transition(action as TAction, context));
    };
}
```

Here, we dequeue an action, then pass it to the given transition.

In many cases, we expect multiple players to take the same action, for example
each player picks between rock, paper, or scissors - in this case, we will
expect one remote action coming in from each player (including us), of the same
type. Most times we want to treat these actions the same way, which means we
want to run the same `Transition` function for each. The `repeat()` function
takes a `RunnableTransition` and repeats it a given number of times:

```typescript
function repeat<TContext>(
    transition: RunnableTransition<TContext>,
    times: number
): RunnableTransition<TContext>[] {
    return Array(times).fill(transition);
}
```

This gives as an array of `RunnableTransitions` we can execute in sequence.

Finally, we might want to combine the output of calling `local()` with the
output of calling `repeat()` into a longer sequence of `RunnableTransitions` we
can run - the first function gives us a `RunnableTransition`, the second
function gives us an array of `RunnableTransition`s. To address this, we
provide `sequence`:

```typescript
function sequence<TContext>(
    transitions: (
        | RunnableTransition<TContext>
        | RunnableTransition<TContext>[]
    )[]
): RunnableTransition<TContext>[] {
    return transitions.flat();
}
```

This function takes an array of `RunnableTransition`s, or an array of
arrays, and calls `flat()` on this to flatten nested array into a single, flat
list.

Once we have a sequence of transitions, we can run them using `run()`:

```typescript
async function run<TContext>(
    sequence: RunnableTransition<TContext>[],
    queue: IQueue<BaseAction>,
    context: TContext
) {
    for (const transition of sequence) {
        await transition(queue, context);
    }
}
```

We simply execute each `RunnableTransition` in turn.

Understandably, this has all been abstract. Let's now see how we can use these
functions to model interactions.

## Interactions

Let's look at a simple example: key exchange: in order to secure our transport,
we want each client to share a public key, then sign each subsequent message
with their corresponding private key.

### Key exchange

We looked at securing the transport layer in [this post](https://vladris.com/blog/2023/11/28/mental-poker-part-3-transport.html).
We haven't discussed the key negotiation though.

Let's create the following protocol: as each client joins the game, they post
a public key. For an `N` player game, each client should expect `N` remote
transitions consisting of clients publishing public keys. Once all of these
were processed, we should have all public keys for all clients and can create
a `SignedTransport`.

Let's sketch out the state machine:

```typescript
function makeKeyExchangeSequence(players: number) {
    return sm.sequence([
        sm.local(async (actionQueue: IQueue<KeyExchangeAction>, context: CryptoContext) => {
            // Post public key ...
        }),
        sm.repeat(sm.transition((action: KeyExchangeAction, context: CryptoContext) => {
            // Store incoming public key ...
        }), players)
    ]);
}
```

Note we create a `LocalTransition` in which we post our own public key, and we
repeat the remote transition handling an incoming public key (remember with
Fluid we expect the server to also send us back whatever we post).

Clients can join the game at different times, so we don't know in what order
the keys will come in but, luckily, each `Action` has a `clientId` so we know
who's key it is.

We'll look at the implementation of the transitions but first let's see what
are the `KeyExchangeAction` and `CryptoContext`:

```typescript
type KeyExchangeAction = {
    clientId: ClientId;
    type: "KeyExchange";
    publicKey: Key;
};

type CryptoContext = {
    clientId: ClientId;
    me: PublicPrivateKeyPair;
    keyStore: KeyStore;
};
```

`KeyExchange` is an action consisting of `clientId` and `publicKey`, with the
`type` set to `"KeyExchange"`.

`CryptoContext` is the context needed by the transitions implementing the key
exchange - that is we need to know our own `clientId`, our public-private
key pair, and we need a `keyStore`, which is a map of `clientId` to public key.
We looked at the `KeyStore` and the other key types in a previous blog post, but
here they are again for reference:

```typescript
type Key = string;

type PublicPrivateKeyPair = {
    publicKey: Key;
    privateKey: Key;
};

type KeyStore = Map<ClientId, Key>;
```

With these in place, let's look at the implementation of the transitions:

```typescript
function makeKeyExchangeSequence(players: number) {
    return sm.sequence([
        sm.local(
            async (
                actionQueue: IQueue<KeyExchangeAction>,
                context: CryptoContext
            ) => {
                // Post public key
                await actionQueue.enqueue({
                    type: "KeyExchange",
                    clientId: context.clientId,
                    publicKey: context.me.publicKey,
                });
            }
        ),
        sm.repeat(
            sm.transition(
                (action: KeyExchangeAction, context: CryptoContext) => {
                    // This should be a KeyExchangeAction
                    if (action.type !== "KeyExchange") {
                        throw new Error("Invalid action type");
                    }

                    // Protocol expects clients to post an ID
                    if (action.clientId === undefined) {
                        throw new Error("Expected client ID");
                    }

                    // Protocol expects each client to only post once and to have a unique ID
                    if (context.keyStore.has(action.clientId)) {
                        throw new Error(
                            "Same client posted key multiple times"
                        );
                    }

                    context.keyStore.set(action.clientId, action.publicKey);
                }
            ),
            players
        ),
    ]);
}
```

`sm` stands for "state machine". The functions described above live in a
`StateMachine` namespace aliased to `sm`.

Our local transition is simple: we enqueue a `KeyExchangeAction`, sending our
`clientId` and `publicKey` from the `CryptoContext`.

When a remote action comes in, we perform the required validations:

* Ensure it is a `KeyExchangeAction`.
* Ensure it has a `clinetId`.
* Ensure the same client doesn't post two different public keys.

Finally, we store the `clientId` and `publicKey`.

The end-to-end implementation for key exchange, relying on the state machine,
is here:

```typescript
async function makeCryptoContext(clientId: ClientId): Promise<CryptoContext> {
    return {
        clientId,
        me: await Signing.generatePublicPrivateKeyPair(),
        keyStore: new Map<ClientId, Key>(),
    };
}

async function keyExchange(
    players: number,
    clientId: ClientId,
    actionQueue: IQueue<BaseAction>
) {
    const context = await makeCryptoContext(clientId);

    const keyExchangeSequence = makeKeyExchangeSequence(players);

    await sm.run(keyExchangeSequence, actionQueue, context);

    return [context.me, context.keyStore] as const;
}
```

`makeCryptoContext()` is a helper function to initialize a `CryptoContext`
instance - it takes a `clientId`, generates a public-private key pair, and
initializes an empty key store.

`keyExchange()` calls the functions we defined previously to get a
`CryptoContext`, the key exchange sequence, and calls the state machine's
`run()` to execute the key exchange.

Once done, it returns the client's public-private key pair, and the key store.

From a caller's perspective, the protocol handling key exchange is now
abstracted away behind the `keyExchange()` function. The caller doesn't have to
worry about the mechanics of exchanging keys, rather can just call this and get
back all the required data to create a `SignedTransport`.

### Rock-paper-scissors

As a second example, we'll sketch out the state machine for a game of
rock-paper-scissors. We won't dive into all the implementation details. At a
high level, here is how we play a game of rock-paper-scissors:

* Each player picks from *rock*, *paper*, or *scissors*, encrypts their
  selection, and posts it.
* Once selections are posted, each player posts the key they used to encrypt.

This two-step ensures players are committed to a selection and can't cheat by
observing what the other player picked and picking afterwards.

The state machine for this game is:

* A local transition in which we make our local selection.
* Two remote transitions, getting from the server our selection and the other
  player's.
* A local transition in which we share our encryption key.
* Two remote transitions, getting the encryption keys from the server.

The state machine is:

```typescript
sm.sequence([
    sm.local(async (queue, context) => {
        // Post our play action
    }),
    sm.repeat(sm.transition(async (action, context) => {
        // Both player and opponent need to post their encrypted selection
    }), 2),
    sm.local(async (queue, context) => {
        // Post our reveal action
    }),
    sm.repeat(sm.transition(async (reveal: RevealAction, context: RootStore) => {
        // Both player and opponent need to reveal their selection
    }), 2)
]);
```

We won't fill in the functions in this post but this gives you an idea of how
we can model a more complex set of steps using our library.

## Summary

In this post we looked at a state machine we can use to implement games:

* The state machine needs to be very unopinionated as each game implements
  its own logic, defines its own `Action` types, and has its own relevant
  context.
* Local transitions are functions we initiate locally and they usually end
  with an action being posted.
* Remote transitions are functions we run in response to actions arriving
  from the servers - these could've been originated by us or by another
  client.
* `RunnableTransition` is a common type that can wrap local or remote
  transitions.
* We can combine transitions by repeating them, or concatenating them into
  sequences. Once we have a sequence of transitions, we can run it to
  implement a protocol.
* We saw how key exchange can be implemented on top of a state machine and
  sketched out the state machine for a game of rock-paper-scissors.

The Mental Poker Toolkit is [here](https://github.com/vladris/mental-poker-toolkit).
This post covered the [state-machine package](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/state-machine),
the key exchange is implemented in the [primitives package](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/primitives).

[^1]: <https://en.wikipedia.org/wiki/Finite-state_machine>.

[^2]: *Sequenced* is a [Fluid Framework](https://fluidframework.com/) term.
Clients send messages to the Fluid relay service, which orders them in
the order they came in and broadcasts them to all clients. This is to
ensure all clients eventually see all the messages sent *in the same
order*.
