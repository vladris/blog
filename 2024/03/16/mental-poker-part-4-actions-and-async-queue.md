# Mental Poker Part 4: Actions and Async Queue

As I was building up the library and looking at state machines that would run
turns in a game, I realized an async queue would come in handy. The challenge
with the raw `ITransport` interface built on top of the Fluid ledger is that if
you are not the first client to join a session, you end up with a set of ops
that already exist on the ledger. You need a way to consume both the ops that
were already sequenced and new incoming ops. An async interface is also easier
to consume than callbacks.

Before diving into that though, let’s talk about actions.

## Actions

As a reminder, *op* is the Fluid Framework term for data being sent/received. In
Mental Poker we use *actions*. All actions should be subtypes of `BaseAction`:

```typescript
export type ClientId = string;

export type BaseAction = {
    clientId: ClientId;
    type: unknown;
};
```

Every action should have a `clientId` showing which client it came from, and a
`type`.

For example, here’s how we would model a game of Rock/Paper/Scissors:

* Both players pick *rock* or *paper* or *scissors*, encrypt their selection,
  and post it on the ledger.
* Next, both players post their encryption key, so the other player can decrypt
  and see the selection.

We model the game in these two steps so regardless of which player moves first,
the player choices are revealed after they have been put on the ledger. If a
player would simply post their unencrypted selection, the other player might
cheat by looking at it before posting their own.

I will cover the Rock/Paper/Scissors implementation in detail in a future post,
for now, let’s just go over the actions:

```typescript
export type PlayAction = {
    clientId: ClientId;
    type: "PlayAction";
    encryptedSelection: EncryptedSelection;
};

export type RevealAction = {
    clientId: ClientId;
    type: "RevealAction";
    key: SerializedSRAKeyPair;
};

export type Action = PlayAction | RevealAction;
```

The two actions described above are modeled as `PlayAction` and `RevealAction`.
Both of these have a `clientId` and `type`, thus are subtypes of `BaseAction`.
Finally, the `Action` type represents all possible actions in the game.

This becomes relevant as we move higher in the stack of the Mental Poker
library. Once we start encoding some of the game semantics, we require generic
types to extend `BaseAction`. This is what happens with the async queue.

## Async queue

As I mentioned at the beginning of the article, queues aim to provide a nicer
API over the transport. The interface is very simple:

```typescript
export interface IQueue<T extends BaseAction> {
    enqueue(value: T): Promise<void>;

    dequeue(): Promise<T>;
}
```

For any type `T` extending `BaseAction`, we can `enqueue()`  a value and we can
`dequeue()` a value. Both of the operations are asynchronous.

I’ll show the full implementation then go over the details:

```typescript
export class ActionQueue<T extends BaseAction> implements IQueue<T> {
    private queue: T[] = [];

    constructor(
        private readonly transport: ITransport<T>,
        preseed: boolean = false
    ) {
        transport.on("actionPosted", (value) => {
            this.queue.push(value);
        });

        if (preseed) {
            for (const value of transport.getActions()) {
                this.queue.push(value);
            }
        }
    }

    async enqueue(value: T) {
        await this.transport.postAction(value);
    }

    async dequeue(): Promise<T> {
        const result = this.queue.shift();
        if (result) {
            return Promise.resolve(result);
        }

        return new Promise<T>((resolve) => {
            this.transport.once("actionPosted", async () => {
                resolve(await this.dequeue());
            });
        });
    }
}
```

The implementation maintains an array of `T`s (actions). The constructor takes a
`transport` argument of type `ITransport` and `preseed` flag:

```typescript
constructor(
    private readonly transport: ITransport<T>,
    preseed: boolean = false
) {
    transport.on("actionPosted", (value) => {
        this.queue.push(value);
    });

    if (preseed) {
        for (const value of transport.getActions()) {
            this.queue.push(value);
        }
    }
}

/* ... */
```

The queue starts listening to the `actionPosted` event and whenever we have an
incoming value, we push it to the internal queue. If `preseed` is `true`, we
also push all actions already posted to the queue.

The reason we make this optional is that we might end up using multiple queues
in a game implementation but we only want to consume the actions posted on the
ledger before we joined the session once. After we are “up to speed”, new
incoming actions fire events which we can consume in realtime. So we would
usually create our first queue with `preseed` set to `true` and subsequent
queues with `preseed` set to `false`.

Enqueuing a value is trivial - we leverage the transport’s `postAction` API:

```typescript
/* ... */

async enqueue(value: T) {
    await this.transport.postAction(value);
}

/* ... */
```

Dequeuing is a bit more interesting:

```typescript
/* ... */

async dequeue(): Promise<T> {
    const result = this.queue.shift();
    if (result) {
        return Promise.resolve(result);
    }

    return new Promise<T>((resolve) => {
        this.transport.once("actionPosted", async () => {
            resolve(await this.dequeue());
        });
    });
}

/* ... */
```

First, we call `shift()` on the queue. This either returns a value or
`undefined` if the queue is empty.

If we do get a value, we return a resolved promise right away.

If we don’t have a value, we add a one-time listener to the `actionPosted`
event. When a new action is posted, the underlying transport will fire the
event. Since event listeners are called in the order they subscribed, we are
guaranteed the listener we added in the constructor fires first, and adds the
value to `queue`. We resolve the promise by recursively calling `dequeue()` and
awaiting the response.

The reason we do this is we might have multiple callers to `dequeue()` holding
on to promises. In this case, we don’t want to resolve all of them with the
incoming value, rather just the first one. The first recursive call to
`dequeue()` should grab the value from the internal `queue` and return it right
away, while other recursive callers would end up awaiting again until a new
value comes in. There's probably a more efficient non-recursive implementation
but for our specific use-case (games), we don't expect many cases where we have
multiple dequeus pending.

## Using the queue

There are two main reasons for using this queue rather than relying directly on
the underlying transport.

First, the underlying transport can have a set of actions (messages) that
already arrived on the client (which we would retrieve with the `getActions()`
method), and some which arrive in real time (which would fire events). The
queue gives us a unified way to consume both, by calling `await dequeue()`.

Besides a unified interface, we expect multiple spots in the code to wait for
an incoming action. This depends on the game implementation, but usually at
different game states we expect different messages to come in. This is harder
to achieve waiting for event callbacks and much easier to do via the same
`await dequeue()` call.

## Summary

In this post we looked at *actions*, the key building blocks of Mental Poker
games, and an async queue which provides a clean abstraction over the underlying
transport.

The code covered in this post is available on GitHub in
the **[mental-poker-toolkit repo](https://github.com/vladris/mental-poker-toolkit/)**.
`BaseAction` and the `ITransport` and `IQueue` interfaces are part of the core
types package **[packages/types](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/types)**.
`ActionQueue` is implemented under **[packages/action-queue](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/action-queue)**.
