# Mental Poker Part 8: Rock-Paper-Scissors

For an overview on Mental Poker, see **[Mental Poker Part 0: An
Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html)**.
Other articles in this
series: **[https://vladris.com/writings/index.html\#mental-poker](https://vladris.com/writings/index.html#mental-poker)**.
In the previous **[post in the
series](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html)** we
looked at some low-level building blocks. It this post, we’ll finally see how to
implement a game end-to-end using the toolkit. We’ll start with a simple game:
rock-paper-scissors.

## Overview

We’ll build this game as a React app, using the toolkit. We’ll be using
[Redux](https://redux.js.org/) for state management - Redux provides a good way
of binding game state to the UI, which works well with our toolkit.

The full code for this is in the
[`demos/rock-paper-scissors`](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/rock-paper-scissors)
app.

## Model

Since we got a lot of the primitives out of the way in the previous post (Fluid
connection, getting a `SignedTransport` etc.), in this post we can focus on the
higher level semantics of modeling the game.

We’ll play a round of rock-paper-scissors as follows:

* Both players post their selection (`rock` or `paper` or `scissors`) encrypted.
* Both players reveal their decryption key.

This 2-step protects against cheating: before the game proceeds, both players
need to make a selection. But the other player doesn’t know what the selection
is until the decryption key is provided. Note for this particular game, turn
order doesn’t matter.

We’ll start with a few type definitions:

```typescript
type PlaySelection = "Rock" | "Paper" | "Scissors";

type EncryptedSelection = string;
```

`PlaySelection` represents the possible plays, `EncryptedSelection` is the
string representation of an encrypted `PlaySelection`.

Our game model will have 2 actions:

```typescript
type PlayAction = {
    clientId: ClientId;
    type: "PlayAction";
    encryptedSelection: EncryptedSelection;
};

type RevealAction = {
    clientId: ClientId;
    type: "RevealAction";
    key: SerializedSRAKeyPair;
};

type Action = PlayAction | RevealAction;
```

`PlayAction` is the first step, when players post their encrypted choice.
`RevealAction` is the second step, revealing the encryption key. We’ll use the
SRA algorithm for encryption since we have it in our toolkit, but for this game
any encryption algorithm would work.

We’ll also need a couple more type definitions for the game state:

```typescript
type GameStatus = "Waiting" | "Ready" | "Win" | "Loss" | "Draw";

type PlayValue =
    | { type: "Selection"; value: PlaySelection }
    | { type: "Encrypted"; value: EncryptedSelection }
    | { type: "None"; value: undefined };
```

The `GameStatus` represents the different states the client can be in:

* `Waiting` for another player to connect or for round to finish.
* `Ready` to play.
* `Win`, `Loss`, `Draw` - the result after playing a round.

The `PlayValue` represents the current state of a player’s pick. It can be
either an encrypted selection, a revealed selection, or nothing (at the start of
the game).

Before implementing the game state machine, let’s look at the Redux store.

## Store

I won’t go into the details of Redux in this post - please refer to the
[Redux](https://redux.js.org/) documentation for that. We’ll be using the [Redux
Toolkit](https://redux-toolkit.js.org/) to streamline setting up our store.

We will maintain 6 pieces of state:

* Our ID.
* The other player’s ID.
* The Mental Poker async queue we implement the game on top of.
* The game status (`GameStatus` above).
* Our play (`PlayValue` above).
* The other player’s play (also a `PlayValue`).

We’ll use the Redux Toolkit `createAction` helper to define the update functions
for these:

```typescript
const updateId = createAction<string>("id/update");
const updateOtherPlayer = createAction<string>("otherPlayer/update");
const updateQueue = createAction<IQueue<Action>>("queue/update");
const updateGameStatus = createAction<GameStatus>("gameStatus/update");
const updateMyPlay = createAction<PlayValue>("myPlay/update");
const updateTheirPlay = createAction<PlayValue>("theirPlay/update");
```

We’ll also need reducers (another Redux concept) for updating the values. We can implement a helper function to create these:

```typescript
function makeUpdateReducer<T>(
    initialValue: T,
    updateAction: ReturnType<typeof createAction>
) {
    return createReducer({ value: initialValue }, (builder) => {
        builder.addCase(updateAction, (state, action) => {
            state.value = action.payload;
        });
    });
}
```

Finally, we set up our Redux store as:

```typescript
const store = configureStore({
    reducer: {
        id: makeUpdateReducer("", updateId),
        otherPlayer: makeUpdateReducer("Not joined", updateOtherPlayer),
        queue: makeUpdateReducer<IQueue<Action> | undefined>(
            undefined,
            updateQueue
        ),
        myPlay: makeUpdateReducer<PlayValue>(
            { type: "None", value: undefined },
            updateMyPlay
        ),
        theirPlay: makeUpdateReducer<PlayValue>(
            { type: "None", value: undefined },
            updateTheirPlay
        ),
        gameStatus: makeUpdateReducer("Waiting", updateGameStatus),
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware({
            serializableCheck: false,
        }),
});
```

We initialize the store with the default values:

* We don’t have an ID.
* The other player hasn’t joined yet.
* We don’t have an async queue.
* Neither player has any play.
* The game state is `Waiting` (for other player to connect).

That’s about it for Redux setup - again, I won’t cover what reducers are, how
Redux manages state changes etc.

## Playing a round

We’ll implement playing a round of rock-paper-scissors in the function `async
function playRound(selection: PlaySelection)`. We invoke this with our selection
(rock, paper, or scissors).

First, we need to get a few references:

```typescript
const context = store;

await context.dispatch(updateGameStatus("Waiting"));

const queue = context.getState().queue.value!;

const kp = SRA.genereateKeyPair(BigIntUtils.randPrime());
```

First, we get a reference to the Redux store. Then we update the game status to
`Waiting`. We get a reference to the async queue from the Redux store and,
finally, we generate an SRA key pair. The `generateKeyPair()` and `randPrime()`
functions we discussed all the way in [part
1](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html),
when we covered cryptography. The `dispatch()` and `getState()` are standard
Redux calls.

Now let’s look at the state machine modeling a round. It consists of the
following sequence:

1. Post our encrypted selection.
2. Expect to receive 2 encrypted selections (ours and the opponent’s).
3. Post our encryption key to reveal our selection.
4. Expect to receive 2 encryption keys (ours and the opponent’s).

We can run this state machine with the Redux store as context:

```typescript
await sm.run(sm.sequence([
        sm.local(async (queue) => {
            const playAction = {
                clientId: context.getState().id.value,
                type: "PlayAction",
                encryptedSelection: SRA.encryptString(selection, kp),
            };

            await queue.enqueue(playAction);
        }),
        sm.repeat(sm.transition(async (play: PlayAction, context: RootStore) => {
            const action =
            play.clientId === context.getState().id.value
                ? updateMyPlay
                : updateTheirPlay;

            await context.dispatch(
                action({ type: "Encrypted", value: play.encryptedSelection })
        );
        }), 2),
        sm.local(async (queue) => {
            const revealAction = {
                clientId: context.getState().id.value,
                type: "RevealAction",
                key: SRAKeySerializationHelper.serializeSRAKeyPair(kp),
            };
            
            await queue.enqueue(revealAction);
        }),
        sm.repeat(sm.transition(async (reveal: RevealAction, context: RootStore) => {
            const action =
                reveal.clientId === context.getState().id.value
                    ? updateMyPlay
                    : updateTheirPlay;
            const originalValue =
                reveal.clientId === context.getState().id.value
                    ? context.getState().myPlay.value
                    : context.getState().theirPlay.value;

            await context.dispatch(
                action({
                    type: "Selection",
                    value: SRA.decryptString(
                        originalValue.value as EncryptedSelection,
                        SRAKeySerializationHelper.deserializeSRAKeyPair(reveal.key)
                    ) as PlaySelection,
                })
            );
        }), 2)
    ]), queue, context);
```

We first define a `local` transition - we enqueue our `PlayAction`.

We then repeat 2 times a `transition`. We update the Redux store accordingly: if
the received client ID is ours, we call `updateMyPlay()`, otherwise we call
`updateTheirPlay()` with the encrypted value.

Next, we enqueue our `RevealAction`.

We then again repeat 2 times a `transition`. If the incoming client ID is ours,
we call `updateMyPlay()` and decrypt the `originalValue` (`myPlay.value`) with
the received key, otherwise we call `updateTheirPlay()` and decrypt the
`originalValue` (`theirPlay.value`) with the received key.

Note how this code updates the Redux store directly, by using it as the context
for the state machine.

Once the state machine finishes, we should have both our play and the opponent’s
play, so we can determine the winner and update the game state accordingly:

```typescript
const myPlay = context.getState().myPlay.value;
const theirPlay = context.getState().theirPlay.value;

if (myPlay.value === theirPlay.value) {
    await context.dispatch(updateGameStatus("Draw"));
} else if (
    (myPlay.value === "Rock" && theirPlay.value === "Scissors") ||
    (myPlay.value === "Paper" && theirPlay.value === "Rock") ||
    (myPlay.value === "Scissors" && theirPlay.value === "Paper")
) {
    await context.dispatch(updateGameStatus("Win"));
} else {
    await context.dispatch(updateGameStatus("Loss"));
}
```

And that’s it in terms of game mechanics. Finally, let’s look at a simple UI for
the game.

## UI

We’ll build the UI using React. First, let’s create a component that provides
the rock-paper-scissors options as 3 buttons:

```typescript
type ButtonsViewProps = {
    disabled: boolean;
    onPlay: (play: PlaySelection) => void;
}

const ButtonsView = ({ disabled, onPlay }: ButtonsViewProps) => {
    return <div>
        <button disabled={disabled} onClick={() => onPlay("Rock")} style={{ width: 200}}>🪨</button>
        <button disabled={disabled} onClick={() => onPlay("Paper")} style={{ width: 200 }}>📄</button>
        <button disabled={disabled} onClick={() => onPlay("Scissors")} style={{ width: 200 }}>✂️</button>
    </div>
}
```

Our properties are a boolean that determines whether buttons should be enabled
or disabled and an `onPlay()` callback.

Our view is also very simple:

```typescript
const useSelector: TypedUseSelectorHook<RootState> = useReduxSelector;

const MainView = () => {
    const idSelector = useSelector((state) => state.id);
    const otherPlayer = useSelector((state) => state.otherPlayer);
    const gameStateSelector = useSelector((state) => state.gameStatus);

    return <div>
        <div>
        <p>Id: {idSelector.value}</p>
        <p>Other player: {otherPlayer.value}</p>
        <p>Status: {gameStateSelector.value}</p>
        </div>
        <ButtonsView disabled={gameStateSelector.value === "Waiting"} onPlay={playRound}></ButtonsView>
    </div>
}
```

The first line is some React-Redux plumbing (via the `react-redux` package),
which allows us to grab data from the Redux store and put it in the UI.

We’ll be showing our ID, the other player’s ID, the game status, and the 3
buttons. The buttons are enabled as long as the game state is no `Waiting`. Once
the user clicks a button, we simply call the `playRound()` function we looked at
in the previous section.

Rendering all of this on the page:

```typescript
const root = ReactDOM.createRoot(document.getElementById("root")!);
root.render(
    <Provider store={store}>
        <MainView />
    </Provider>
);
```

Here, `Provider` comes from the `react-redux` package and makes the Redux store
available to the React components.

## Initialization

We now have all the pieces into place, the only bit of code we haven’t covered
is initializing the game:

```typescript
getLedger<Action>().then(async (ledger) => {
    const id = randomClientId();

    await store.dispatch(updateId(id));

    const queue = await upgradeTransport(2, id, ledger);
    
    await store.dispatch(updateQueue(queue));

    for (const action of ledger.getActions()) {
        if (action.clientId !== id) {
            store.dispatch(updateOtherPlayer(action.clientId));
            break;
        }
    }

    await store.dispatch(updateGameStatus("Ready"));
});
```

The steps are:

* We connect to the Fluid session and get a reference to the `ledger`, as we saw
  in the [previous
  post](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html).
* We generate a random client ID (I’m not covering the `randomClientId()`
  function in this post, but you can find the implementation in
  [`packages/primitives/src/randomClientId.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/primitives/src/randomClientId.ts)).
* Update our ID in the Redux store.
* We call `upgradeTransport()` (also discussed in the [previous
  post](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html)).
* Update the Redux store with a reference to the async queue.
* We retrieve and store the other player’s ID.
* We update the game status to `Ready` (from the default, which is `Waiting`).

The steps are pretty self-explanatory, maybe except getting the other player’s
ID. The way that works is as follows: `getActions()` returns all actions posted
on the ledger so far. We look for an action where the client ID is different
than our client ID and store that as the other player’s ID. We are guaranteed to
see at least one action from the other player, as we ran `upgradeTransport()`,
which under the hood performs a public key exchange.

And that’s it - we have an end-to-end game of rock-paper-scissors.

## Summary

We looked at implementing rock-paper-scissors using the Mental Poker toolkit.
The full source code for the demo is under
[`demos/rock-paper-scissors`](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/rock-paper-scissors).

* Instructions on how to run the game in
  [README.md](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/README.md).
* The game model is implemented in
  [`model.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/src/model.ts).
* The Redux store is implemented in
  [`store.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/src/store.ts).
* The two React components are
* [`buttonsView.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/src/buttonsView.tsx)
  and
  [`mainView.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/src/mainView.tsx).
* Initialization happens in
  [`index.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/rock-paper-scissors/src/index.tsx).

Note how easy it is to model a game if we rely on the toolkit’s primitives. We
implement the game logic in the model, relying on the toolkit’s capabilities. We
use Redux to store game state, which we can easily bind to a React view. That
said, this was a very simple game. In the next post we’ll look at implementing a
card game.
