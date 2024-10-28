# Mental Poker Part 10: Conclusions

This blog post will wrap up the [Mental Poker
series](https://vladris.com/writings/index.html#mental-poker). I started
thinking about this 3 years ago, and worked on a [Mental Poker
Toolkit](https://github.com/vladris/mental-poker-toolkit) library as a
side-project. The blog posts in the series were written as I was exploring the
tech. Here I aim to bring all the learnings together in a final recap.

## Inception

This all started with [Fluid Framework](https://fluidframework.com/). As the
team was building out the framework, we used hackathons to implement various
applications of Fluid. Since Fluid powers real time collaboration, team members
came up with all sorts of ideas. For example, when I joined the team, I built a
simple collaborative coloring app where multiple clients can simultaneously
color a black and white drawing. A recurring theme was games - building
multiplayer games on top of the framework. The challenge with building games is
hiding information. In Fluid, all data is synchronized to all clients and there
is no central authority. The [Azure Fluid
Relay](https://learn.microsoft.com/en-us/azure/azure-fluid-relay/) isn’t running
app code, so there isn’t an easy way to maintain hidden state for a game (e.g.
cards in hand).

I was looking for a way to do this and learned about [mental
poker](https://en.m.wikipedia.org/wiki/Mental_poker). Mental Poker is a way to
play games with private information in a zero-trust environment, without relying
on a central authority to, for example, deal cards. This is a good fit for
Fluid. As a side-project, I decided to build a library to enable development of
this type of games that would work with Fluid as the underlying communication
mechanism.

So how do players agree on which cards they are dealt, without knowing their
opponent's hand?

## Cryptography

The first big piece I covered was cryptography. Mental Poker relies on
commutative encryption but most commonly used encryption algorithms are
non-commutative. Commutative here meaning that if both Alice and Bob encrypt
something with their keys, it doesn't matter the order in which they apply their
keys to decrypt.

Since I couldn't find a library that provides a symmetric encryption algorithm,
I implemented the SRA algorithm (SRA, not RSA - same people's initials,
different algorithm). Also ended up implementing a bunch of BigInt math, all
covered in [Mental Poker Part 1:
Cryptography](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html).
The blog post covers in detail how shuffling a deck of cards works and what are
the cryptography primitives used.

## Ledger

Next, looking at game modeling, I decided a good way to represent a turn-based
game is an append-only list. Each game step is a node in the list.

Fluid Framework relies on Distributed Data Structures (DDSes) to maintain state
and synchronize it across clients. I implemented this ledger as a Fluid
Framework DDS [here](https://github.com/vladris/fluid-ledger). This is outside
of the [mental-poker-toolkit
repo](https://github.com/vladris/mental-poker-toolkit), since it is generally
useful outside of Mental Poker.

The DDS is the lowest-level representation of a game. I covered this in [Mental
Poker Part 2: Fluid
Ledger](https://vladris.com/blog/2023/06/04/mental-poker-part-2-fluid-ledger.html).

## Transport

Wrapping up the plumbing, I looked as a simple abstraction over the transport
layer. This is a very simple interface:

```typescript
// Transport interface
export declare interface ITransport<T> {
    // Get all the actions that have been posted so far
    getActions(): IterableIterator<T>;

    // Post an action
    postAction(value: T): Promise<void>;

    // Event emitter
    once(event: "actionPosted", listener: (value: T) => void): this;
    on(event: "actionPosted", listener: (value: T) => void): this;
    off(event: "actionPosted", listener: (value: T) => void): this;
}
```

Here, an *action* is an item on our ledger list. We can get all actions posted
to the ledger so far, post a new action, and hook up event listeners.

Note the interface doesn't mention the ledger, so we can swap implementations if
needed. The toolkit relies on Fluid (the `FluidTransport` implementation of this
interface) but this could be swapped out for something else as long as this
interface is satisfied.

I also implemented a `SignedTransport` as a decorator, which adds signature
verification for an existing `ITransport`. Since there is no central authority
and multiple clients can be part of a session, to mitigate spoofing we want
clients to exchange public keys as a first step, then sign all subsequent
messages with private keys. This a different algorithm than SRA, regular
asymmetric cryptography signing and signature verification. I implemented this
on top of
[`crypto.subtle`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/subtle).

I covered all of this in [Mental Poker Part 3:
Transport](https://vladris.com/blog/2023/11/28/mental-poker-part-3-transport.html).

## Actions

I briefly mentioned *actions* in the **Ledger** section. For the Mental Poker
toolkit, all actions are supposed to contain a `clientID` property, identifying
the client, and a `type`, which is a string literal, used to identify the
action. Plus any additional payload the action might need.

```typescript
export type ClientId = string;

export type BaseAction = {
    clientId: ClientId;
    type: unknown;
};
```

## Async Queue

The async queue is something I haven't considered when starting the project, but
I realized using the `ITransport` interface is cumbersome. While it maps well
over Fluid, using it to implement games is not ergonomic.

The async queue provides a better interface over the transport:

```typescript
export interface IQueue<T extends BaseAction> {
    enqueue(value: T): Promise<void>;

    dequeue(): Promise<T>;
}
```

The implementation itself is fairly straightforward, relying on the `ITransport`
APIs and events. With this, clients can enqueue and dequeue actions and await on
the response.

Both actions and the queue implementation are covered in [Mental Poker Part 4:
Actions and Async
Queue](https://vladris.com/blog/2024/03/16/mental-poker-part-4-actions-and-async-queue.html).

Note that by now, running a game using the toolkit can be done by just relying
on actions and the two queue APIs: `enqueue()` and `dequeue()`. Very simple.

## State Machine

Of course, we need a way to model games. Game rules are implemented as sequences
of actions. An action is an atomic step. Note that a game move, for example
drawing a card, doesn't necessarily map to a single action going over the
transport. A game move, especially in the context of Mental Poker, can involve
several steps (actions) taken by the players.

The state machine aims to facilitate game implementation.

### Transitions

I implemented two core state machine pieces: *local transitions* and *remote
transitions*.

A local transition means an action originates on our client. For example the
player decides to discard a card or, in a game of rock-paper-scissors, the
player picks between the 3 options. This means we will run some code and enqueue
an action:

```typescript
type LocalTransition<TAction extends BaseAction, TContext> = (
    actionQueue: IQueue<TAction>,
    context: TContext
) => void | Promise<void>;
```

We take the queue as a parameter. The `context` can be anything, it's a way to
pass additional game state to the function.

A remote transition means we receive an action.

```typescript
type Transition<TAction extends BaseAction, TContext> = (
    action: TAction,
    context: TContext
) => void | Promise<void>;
```

Here, we dequeue an action and invoke the transition, passing the action as an
argument.

We need both of these transitions to implement a game, but we can provide a
unified abstraction:

```typescript
type RunnableTransition<TContext> = {
    actionQueue: IQueue<BaseAction>,
    context: TContext
}: Promise<void>;
```

We can adapt a `Transition` to this type by calling dequeue on the `actionQueue`
and passing the resulting action to the `Transition`.

The state machine takes an array of `RunnableTransition`s and executes the code
in sequence. It also provides several helper functions:

* `local()`, to create `RunnableTransition` from a `LocalTransition`.
* `transition()`, to create a `RunnableTransition` from a (remote) `Transition`.
* `repeat()`, to repeat a given `RunnableTransition` a `number` of times.
* `transitions()`, to convert several `RunnableTransition` or
  `RunnableTransition[]` into a flat array of `RunnableTransition`.

The post [Mental Poker Part 5: State
Machine](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html)
covers the implementation in details and also shows examples of modeling rules
as transitions. Here's a rock-paper-scissors skeleton:

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

## Primitives

We now have all the pieces we need to model games. The toolkit also provides
common primitives - plug & play state machines to be integrated in games.

An example of this is card shuffling. Given a deck of cards, there is a state
machine that shuffles this deck according to the Mental Poker steps and hides
this behind a simple `shuffle()` function.

I cover the details of this in [Mental Poker Part 6: Shuffling
Implementation](https://vladris.com/blog/2024/04/07/mental-poker-part-6-shuffling-implementation.html).

Shuffling cards is the canonical example of Mental Poker, but building a game
requires several other common pieces. A few examples:

* Creating a Fluid transport (abstracting the Fluid container and connection
  setup).
* Enabling signature checking, in other words converting a given (unsigned)
  `ITransport` into a `SignedTransport`.
* Establishing turn order for multiple players and agreeing on a large shared
  prime (required by RSA).

I covered all of these in [Mental Poker Part 7:
Primitives](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html).

All implementation rely on the state machine are expressed as sequences of
transitions.

## Games

Finally, I provided a couple of sample games.

The first is rock-paper-scissors. Rock-paper-scissors is interesting because it
does require some cryptography, but it is much simpler than a card game. Players
simply pick between rock, paper, or scissors, encrypt their choice, then post it
(enqueue it). Once both players shared their pick, they share a key the other
player can use to decrypt their pick. Then we can see who won the game.

The implementation is covered in [Mental Poker Part 8:
Rock-Paper-Scissors](https://vladris.com/blog/2024/06/24/mental-poker-part-8-rock-paper-scissors.html).

Next, I implemented a more complex game: discard. In this game, players take
turns discarding cards as long as they can match the value or suit on top of the
discard pile. If they can't discard, they draw a card instead. The first player
to discard their whole hand wins. This is again a fairly simple game in terms of
rules, but requires more advance semantics like card shuffling, drawing and
discarding cards etc.

The implementation is covered in [Mental Poker Part 9: Discard
Game](https://vladris.com/blog/2024/07/18/mental-poker-part-9-discard-game.html).

## Zero-Trust

Mental Poker enables us to play games in a zero-trust environment without a
centralized authority. Of course, there are some limitations.

Signature verification mitigates spoofing, but there is no way to guarantee
other clients aren't colluding over a secondary channel. This isn't a limitation
of Mental Poker, rather in general - even if we play poker with a server
handling the deal, players can cheat and talk to each other with a separate app.

Cryptography ensures certain type of cheating is impossible. For example in the
rock-paper-scissors example, a player can't pretend they picked something else
once their encrypted pick was shared with the other player. Similarly,
cryptography enables maintaining private state over a public channel, including
card shuffling, cards in hand etc.

The state machine helps model games as a sequence of steps. As long as the
clients agree on the rules and follow the steps, they can play a game. Once a
player posts an action that the other player doesn't expect, in other words is
not correct according to the game semantics, the other player can tell the game
rules are not respected. That said, there is no simple way to recover from this.
I call this the "flip the table" recourse. You can't really do much, since
there's no central authority to arbitrate this, but cryptography and the state
machine make it easy for you to tell when another player is cheating and, at the
very least, you can refuse to continue playing.

This was a very fun side-project I worked on, intermittently, for 3 years. I
learned a lot about Mental Poker and built a reusable toolkit for this type of
games. All code discussed in the series is available on GitHub:
<https://github.com/vladris/mental-poker-toolkit/>.
