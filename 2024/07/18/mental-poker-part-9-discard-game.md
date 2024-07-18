# Mental Poker Part 9: Discard Game

For an overview on Mental Poker, see [Mental Poker Part 0: An
Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html).
Other articles in this
series: **[https://vladris.com/writings/index.html\#mental-poker](https://vladris.com/writings/index.html#mental-poker)**.
In the previous [post in the series](https://vladris.com/blog/2024/06/24/mental-poker-part-8-rock-paper-scissors.html)
we looked at building a simple game of rock-paper-scissors. In this post we'll
look at implementing a card game.

## Overview

We'll build a discard game - players take turns discarding a card that must
match either the suit or the value of the card on top of the discard pile. The
player who discards their whole hand first wins.

We're implementing a simple game as the focus is not on the game-specific logic,
rather how to leverage the Mental Poker toolkit.

The full code for this is in the
[`demos/discard`](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/discard)
app. The best way to read this post is side by side with the code.

We'll follow a similar structure to the rock-paper-scissors game we looked at in
the [previous
post](https://vladris.com/blog/2024/06/24/mental-poker-part-8-rock-paper-scissors.htm):

* A *model* implementing the game logic.
* A Redux *store* maintaining game state.
* A React *UI* bound to the store.

## Model

First, let's look at how we implement the deck of cards and associated logic.

### Deck

We'll represent a card as a string, for example `"9:hearts"` is the 9 of hearts.
The function `getDeck()` initializes as unshuffled deck of cards:

```typescript
function getDeck() {
    const deck: string[] = [];

    for (const value of ["9", "10", "J", "Q", "K", "A"]) {
        for (const suit of ["hearts", "diamonds", "clubs", "spades"]) {
            deck.push(value + ":" + suit);
        }
    }

    return deck;
}
```

We're using fewer cards (from 9 to Aces) for this demo as the more cards we have
the more prime numbers we need to find to encrypt them and it slows things down.
Rather than implementing some loading UI, we'll just use fewer cards for the
example.

We need a helper function to tell us whether two cards match either value or
suit:

```typescript
function matchSuitOrValue(a: string, b: string) {
    const [aValue, aSuit] = a.split(":");
    const [bValue, bSuit] = b.split(":");

    return aValue === bValue || aSuit === bSuit;
}
```

Finally, we want a class to wrap a deck and implement the functions needed for
using it:

```typescript
class Deck {
    private myCards: number[] = [];
    private othersCards: number[] = [];
    private drawPile: number[] = [];
    private discardPile: number[] = [];

    private decryptedCards: (string | undefined)[] = [];
    private othersKeys: SRAKeyPair[] = [];

    constructor(
        private encryptedCards: string[],
        private myKeys: SRAKeyPair[],
        private store: RootStore
    ) {
        this.drawPile = encryptedCards.map((_, i) => i);
    }
...
```

We initialize the class with an array of encrypted cards (the shuffled deck) as
`encryptedCards`, our set of SRA keys (`myKeys`) and the Redux store (`store`).

We also need to track cards (by index):

* The cards in our hand (`myCards`).
* The cards in the other player's hand (`othersCards`).
* The draw pile (`drawPile`).
* The discard pile (`discardPile`).

As the other player shares their encryption keys (when they reveal a card to
us), we'll store them in the `othersKeys` array. Similarly, as we decrypt cards,
we'll store them in `decryptedCards` - this is just for convenience, so we don't
have to keep decrypting the same values over and over.

We assume we're starting with a shuffled deck of cards as a draw pile, with no
player having cards in hand - so we initialize `drawPile` to the indexes of
`encryptedCards`.

Some helper functions:

```typescript
    ...
    
    getKey(index: number) {
        return SRAKeySerializationHelper.serializeSRAKeyPair(
            this.myKeys[index]
        );
    }

    getKeyFromHand(index: number) {
        return SRAKeySerializationHelper.serializeSRAKeyPair(
            this.myKeys[this.myCards[index]]
        );
    }

    cardAt(index: number) {
        if (!this.decryptedCards[index]) {
            const partial = SRA.decryptString(
                this.encryptedCards[index],
                this.myKeys[index]
            );

            this.decryptedCards[index] = SRA.decryptString(
                partial,
                this.othersKeys[index]
            );
        }

        return this.decryptedCards[index]!;
    }

    getDrawIndex() {
        return this.drawPile[0];
    }
    
    canIMove() {
        if (this.discardPile.length === 0) {
            return true;
        }

        return (
            this.drawPile.length > 0 ||
            this.myCards.some((index) =>
                matchSuitOrValue(
                    this.cardAt(index),
                    this.cardAt(this.discardPile[this.discardPile.length - 1])
                )
            )
        );
    }
    ...
```

These are pretty self-explanatory:

* `getKey()` returns our SRA key at `index`.
* `getKeyFromHand()` returns our SRA key for a card in our hand (at `index`).
* `cardAt()` returns the decrypted card at `index`. This assumes we can decrypt
  the card. If we are already storing it in `decryptedCards`, we return it from
  there; otherwise we decrypt it using our key and the other player's key, then
  store it in `decryptedCards`.
* `getDrawIndex()` returns the index at the top of the discard pile.
* `canIMove()` returns `true` if we can discard a card. If the discard pile is
  empty, we can discard anything; otherwise at least one of the cards in our
  hand needs to match the suit or value of the card on top of the discard pile.

We also need to implement some functions that mutate the deck (in which case we
also need to update our view-model so our UI reflects the changes):

```typescript
    ...
    async myDraw(serializedSRAKeyPair: SerializedSRAKeyPair) {
        const index = this.drawPile.shift()!;
        this.myCards.push(index);
        this.othersKeys[index] =
            SRAKeySerializationHelper.deserializeSRAKeyPair(
                serializedSRAKeyPair
            );

        await this.updateViewModel();
    }

    async othersDraw() {
        this.othersCards.push(this.drawPile.shift()!);

        await this.updateViewModel();
    }

    async myDiscard(index: number) {
        const cardIndex = this.myCards.splice(index, 1)[0];
        this.discardPile.push(cardIndex);

        this.updateViewModel();
    }

    async othersDiscard(
        index: number,
        serializedSRAKeyPair: SerializedSRAKeyPair
    ) {
        const cardIndex = this.othersCards.splice(index, 1)[0];
        this.othersKeys[cardIndex] =
            SRAKeySerializationHelper.deserializeSRAKeyPair(
                serializedSRAKeyPair
            );
        this.discardPile.push(cardIndex);

        this.updateViewModel();
    }
    ...
```

The actions are:

* `myDraw()` - we draw a card from the top of the draw pile. We need the other
  player's key for this card, given as the `serializedSRAKeyPair` argument.
* `othersDraw()` - other player draws a card from the top of the draw pile. Note
  the `Deck` class just maintains state, so is not responsible for sharing our
  key for that card with the other player - rather we just update the state
  (`othersCards` and `drawPile`).
* `myDiscard()` - we discard a card. We take the `index` of the card as an
  argument.
* `othersDiscard()` - other player discards a card. We take the `index` of the
  card and the other player's SRA key as arguments.

Note all these functions end up calling `updateViewModel()`. That's because all
of the functions change state, so we need to update our Redux store and reflect
the changes on the UI:

```typescript
    ...
    private async updateViewModel() {
        await this.store.dispatch(
            updateDeckViewModel({
                drawPile: this.drawPile.length,
                discardPile: this.discardPile.map((i) => this.cardAt(i)),
                myCards: this.myCards.map((i) => this.cardAt(i)),
                othersHand: this.othersCards.length,
            })
        );
    }
}
```

We haven't looked at the Redux `store` yet. We'll cover this later on but here
we dispatch a deck view-model update. The deck view-model contains the size of
the draw pile, the cards in the discard pile and our hand, and the number of
cards in the other player's hand.

```typescript
type DeckViewModel = {
    drawPile: number;
    discardPile: string[];
    myCards: string[];
    othersHand: number;
};

const defaultDeckViewModel: DeckViewModel = {
    drawPile: 0,
    discardPile: [],
    myCards: [],
    othersHand: 0,
};
```

These is all the deck management logic we need. Let's move on to game *actions*.

### Dealing

We'll be using the library-provided shuffle. We covered this in [part
6](https://vladris.com/blog/2024/04/07/mental-poker-part-6-shuffling-implementation.html)
so we won't go over it again. This is exposed by as a `shuffle()` function. So
assuming our deck is shuffled, the first action we need to handle is *dealing*
cards. In Mental Poker, dealing a card to Bob means Alice needs to share her key
to that card. Then Bob can use his key and Alice's key to "see" the card, while
Alice cannot see it since she doesn't have Bob's key. This is the equivalent of
Bob holding a card in his hand.

We define a `DealAction`:

```typescript
type DealAction = {
    clientId: ClientId;
    type: "DealAction";
    cards: number[];
    keys: SerializedSRAKeyPair[];
}
```

Here, `cards` are the indexes of the cards in the deck and `keys` are the
corresponding SRA keys for each card. Here's the state machine for dealing cards
to both players:

```typescript
async function deal(imFirst: boolean, count: number) {
    const queue = store.getState().queue.value!;

    await store.dispatch(updateGameStatus("Dealing"));

    const cards = new Array(count).fill(0).map((_, i) => imFirst ? i + count : i);
    const keys = cards.map((card) => store.getState().deck.value!.getKey(card)!);

    await sm.run(sm.sequence([
        sm.local(async (queue: IQueue<Action>, context: RootStore) => {
            await queue.enqueue({ 
                clientId: context.getState().id.value, 
                type: "DealAction",
                cards,
                keys });
        }),
        sm.repeat(sm.transition(async (action: DealAction, context: RootStore) => {
            if (action.type !== "DealAction") {
                throw new Error("Invalid action type");
            }

            if (action.clientId === context.getState().id.value) {
                return;
            }

            const deck = context.getState().deck.value!;

            for (let i = 0; i < action.cards.length; i++) {
                if (imFirst) {
                    if (action.cards[i] !== i) {
                        throw new Error("Unexpected card index");
                    }
                    await deck.myDraw(action.keys[i]);
                } else {
                    await deck.othersDraw();
                }
            }

            for (let i = 0; i < action.cards.length; i++) {
                if (imFirst) {
                    await deck.othersDraw();
                } else {
                    if (action.cards[i] !== i + action.cards.length) {
                        throw new Error("Unexpected card index");
                    }
                    await deck.myDraw(action.keys[i]);
                }
            }
        }), 2)
    ]), queue, store);
}
```

In preparation of dealing, we:

* Get the async `queue` from the `store`.
* We update the game status to `Dealing` (more details on this later).
* We determine which cards the *other* player needs - if we're fist, we get the
  first `count` cards so the other player will get the next `count` ones;
  otherwise they get the first `count` cards and we get the next ones.
* We also get the set of keys we need to share with the other player so they can
  decrypt the cards they are dealt.

With this done, our state machine consists of:

* A local transition: we enqueue a `DealAction` with the `cards` and `keys` we
  determined the other player gets.
* A remote transition, repeated twice: we expect to see two `DealAction`
  actions. If we see the one we sent out (the `clientId` matches our `clientId`)
  we can ignore it. If we see the `DealAction` from the other player, we update
  the deck. If we are first to draw, then we call `deck.myDraw()` `count` times,
  then `deck.othersDraw()` `count` times; otherwise we do it the other way
  around - call `deck.othersDraw()` `count` times, then call `deck.myDraw()`
  `count` times.

Local transitions and remote transitions are explained in [part
5](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html),
in which we talked about the state machine.

### Drawing cards

Drawing a card is a two-step process. We need to tell the other player we intend
to draw a card (from the draw pile), and they need to give us their key to that
card. Similarly, if the other player tells us they want to draw a card, we give
them our key to that card.

We need two actions:

```typescript
type DrawRequestAction = {
    clientId: ClientId;
    type: "DrawRequest";
    cardIndex: number;
}

type DrawResponseAction = {
    clientId: ClientId;
    type: "DrawResponse";
    cardIndex: number;
    key: SerializedSRAKeyPair;
}
```

If we want to draw a card, here is our state machine:

```typescript
async function drawCard() {
    const queue = store.getState().queue.value!;

    await store.dispatch(updateGameStatus("Waiting"));

    await sm.run([
        sm.local(async (queue: IQueue<Action>, context: RootStore) => {
            await queue.enqueue({ 
                clientId: context.getState().id.value, 
                type: "DrawRequest",
                cardIndex: context.getState().deck.value!.getDrawIndex() });
        }),
        sm.transition((action: DrawRequestAction) => {
            if (action.type !== "DrawRequest") {
                throw new Error("Invalid action type");
            }
        }),
        sm.transition(async (action: DrawResponseAction, context: RootStore) => {
            if (action.type !== "DrawResponse") {
                throw new Error("Invalid action type");
            }

            await context.getState().deck.value!.myDraw(action.key);
        }),
    ], queue, store);

    await store.dispatch(updateGameStatus("OthersTurn"));
    await waitForOpponent();
}
```

We again get the async `queue` from the `store` and update the game status. Then
we run the state machine consisting of 3 transitions:

* A local transition in which we post our `DrawRequest` action.
* A remote transition in which we expect to see our `DrawRequest`.
* A remote transition in which we expect the other player to respond with a
  `DrawResponse` action, giving us the key and allowing us to draw a card.

Finally, after running the state machine and drawing the card, we update the
game status again to other player's turn and call `waitForOpponent()`, which
we'll cover later.

This fully implements us drawing a card from the top of the discard pile and
updating the deck.

### Discarding cards

Similar to drawing cards, we need to implement discarding cards. Discarding a
card is easier - we don't need a key from the other player, rather we just
provide the key to the card we're discarding such that the other player can
"see" it.

```typescript
type DiscardRequestAction = {
    clientId: ClientId;
    type: "DiscardRequest";
    cardIndex: number;
    key: SerializedSRAKeyPair;
}
```

Our `DiscardRequestAction` contains the card index and our key.

The corresponding state machine:

```typescript
async function discardCard(index: number) {
    const queue = store.getState().queue.value!;

    await store.dispatch(updateGameStatus("Waiting"));

    await sm.run([
        sm.local(async (queue: IQueue<Action>, context: RootStore) => {
            await queue.enqueue({
                clientId: context.getState().id.value, 
                type: "DiscardRequest",
                cardIndex: index,
                key: context.getState().deck.value!.getKeyFromHand(index)});
        }),
        sm.transition(async (action: DiscardRequestAction, context: RootStore) => {
            if (action.type !== "DiscardRequest") {
                throw new Error("Invalid action type");
            }

            await context.getState().deck.value!.myDiscard(action.cardIndex);
        }),
    ], queue, store);

    if (store.getState().deckViewModel.value.myCards.length === 0) {
        await store.dispatch(updateGameStatus("Win"));
    } else {
        await store.dispatch(updateGameStatus("OthersTurn"));
        await waitForOpponent();
    }
}
```

As usual, we get the `queue` and update game state. Then we run the state
machine:

* A local transition posts a `DiscardRequest` with the card index and key.
* A remote transition in which we should see our own `DiscardRequest` - since
  this round-tripped, we can now update the deck.

After running the state machine, we need to check whether we discarded the last
card in our hand. If we did, we can update the game state to us winning.
Otherwise we wait for the other player's move.

### Can't move

The last action we need to look at is the situation in which we can't discard
any card (no matching suit or value) and we also can't draw a card (draw pile is
empty). In this case we lose the game. Since it is our turn, we need to let the
other player know that we're not pondering our next move, rather that we can't
do anything and we lose. We'll model this as a simple `CantMoveAction`:

```typescript
type CantMoveAction = {
    clientId: ClientId;
    type: "CantMove";
}
```

This action has no payload. The state machine is also very simple:

```typescript
async function cantMove() {
    const queue = store.getState().queue.value!;

    await queue.enqueue({ 
        clientId: store.getState().id.value, 
        type: "CantMove" });

    await store.dispatch(updateGameStatus("Loss"));
}
```

At the end of it, we update the game status to us losing.

So far, we have the 3 possible actions we can take when it is our turn:

* Draw a card (via `drawCard()`).
* Discard a card (via `discardCard()`).
* Can't draw, can't discard (via `cantMove()`).

Next, we need to model responding to the other player's move.

### Opponent's turn

The opponent can take the same actions as we can, so we don't need to declare
any new action types, rather we need a state machine that responds to actions
incoming from the other player:

```typescript
async function waitForOpponent() {
    const queue = store.getState().queue.value!;

    const othersAction = await queue.dequeue();

    switch (othersAction.type) {
        case "DrawRequest":
            await sm.run([
                sm.local(async (queue: IQueue<Action>, context: RootStore) => {
                    if (othersAction.cardIndex !== store.getState().deck.value!.getDrawIndex()) {
                        throw new Error("Invalid card index for draw");
                    }

                    await queue.enqueue({
                        clientId: store.getState().id.value,
                        type: "DrawResponse",
                        cardIndex: othersAction.cardIndex,
                        key: store.getState().deck.value!.getKey(othersAction.cardIndex)
                    })}),
                sm.transition(async (action: DrawResponseAction, context: RootStore) => {
                    if (action.type !== "DrawResponse") {
                        throw new Error("Invalid action type");
                    }

                    await context.getState().deck.value!.othersDraw();
                })], queue, store);
            await store.dispatch(updateGameStatus("MyTurn"));
            break;
        case "DiscardRequest":
            await store.getState().deck.value!.othersDiscard(othersAction.cardIndex, othersAction.key);

            if (store.getState().deckViewModel.value.othersHand === 0) {
                await store.dispatch(updateGameStatus("Loss"));
            } else if (store.getState().deck.value?.canIMove()) {
                await store.dispatch(updateGameStatus("MyTurn"));
            } else {
                await cantMove();
            }

            break;
        case "CantMove":
            await store.dispatch(updateGameStatus("Win"));
            break;
        }
}
```

We dequeue an action, then we respond based on its type:

* If this is a `DrawRequest`, we send a `DrawResponse`. We implement this as a
  simple state machine with a local transition (our `DrawResponse`) and a remote
  transition in which we expect to see our response round-tripped. We also check
  to ensure the draw request card index matches the top of the draw pile
  (otherwise the other player might trick us and draw some other card).
* If this is a `DiscardRequest`, we update the deck. If the other player
  discarded their last card, we lose. Otherwise, if we can move, we update game
  status to `MyTurn` and let the user pick which card to discard etc. But if we
  can't move - can't discard anything, can't draw, then we automatically call
  `cantMove()` to mark the fact we lost.
* If this is a `CantMove`, the other player lost so we update game status to
  `Win`.

Note for the discard request, to keep things simple, we aren't checking whether
the move is legal. If we want to secure the implementation, we should check that
the card the other player is discarding matches either the suit or value of the
card on top of the discard pile.

### Actions and status

We already covered all possible actions:

```typescript
type Action = DealAction | DrawRequestAction | DrawResponseAction | DiscardRequestAction | CantMoveAction;
```

The possible game status:

```typescript
type GameStatus = "Waiting" | "Shuffling" | "Dealing" | "MyTurn" | "OthersTurn" | "Win" | "Loss" | "Draw";
```

We just implemented all the game logic - the possible actions a player can take,
and the request/response needed to model the game of discard. We have the full
model, so let's move on to the Redux store.

## Store

Like in the [previous
post](https://vladris.com/blog/2024/06/24/mental-poker-part-8-rock-paper-scissors.html),
we will be using [Redux](https://redux.js.org/) and the [Redux
Toolkit](https://redux-toolkit.js.org/).

The sate we'll be maintaining:

* Our ID.
* The other player's ID.
* The Mental Poker async queue we implement the game on top of.
* The game status (`GameStatus` in our model).
* The deck (represented by an instance of `Deck`).
* The deck view-model (providing just enough data to bind to the UI).

Using `createAction` from the Redux Toolkit:

```typescript
const updateId = createAction<string>("id/update");
const updateOtherPlayer = createAction<string>("otherPlayer/update");
const updateQueue = createAction<IQueue<Action>>("queue/update");
const updateGameStatus = createAction<GameStatus>("gameStatus/update");
const updateDeck = createAction<Deck>("deck/update");
const updateDeckViewModel = createAction<DeckViewModel>("deckViewModel/update");
```

We'll also use the same helper to create Redux reducers as for
rock-paper-scissors:

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

Our Redux store is:

```typescript
const store = configureStore({
    reducer: {
        id: makeUpdateReducer("", updateId),
        otherPlayer: makeUpdateReducer("Not joined", updateOtherPlayer),
        queue: makeUpdateReducer<IQueue<Action> | undefined>(
            undefined,
            updateQueue
        ),
        gameStatus: makeUpdateReducer("Waiting", updateGameStatus),
        deck: makeUpdateReducer<Deck | undefined>(undefined, updateDeck),
        deckViewModel: makeUpdateReducer<DeckViewModel>(defaultDeckViewModel, updateDeckViewModel),
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware({
            serializableCheck: false,
        }),
});
```

This is all we need to connect the model with the view.

## UI

We'll use React. 

### Card

The first component we need is a card:

```typescript
type CardViewProps = {
    card: string | undefined;
    onClick?: () => void;
};

const suiteMap = new Map([
    ["hearts", "♥"],
    ["diamonds", "♦"],
    ["clubs", "♣"],
    ["spades", "♠"]
]);

const CardView: React.FC<CardViewProps> = ({ card, onClick }) => {
    const number = card?.split(":")[0];
    const suite = card ? suiteMap.get(card.split(":")[1]) : undefined;
    const color = suite === "♥" || suite === "♦" ? "red" : "black";

    return <div style={{ width: 70, height: 100, borderColor: "black", borderWidth: 1, borderStyle: "solid", borderRadius: 5, 
                    backgroundColor: card ? "white" : "darkred"}} onClick={onClick}>
        <div style={{ display: card ? "block" : "none", paddingLeft: 15, paddingRight: 15, color }}>
            <p style={{ marginTop: 20, marginBottom: 0, textAlign: "left", fontSize: 25 }}>{number}</p>
            <p style={{ marginTop: 0, textAlign: "right", fontSize: 30 }}>{suite}</p>
        </div>
    </div>
}
```

This renders a `card` which can be a `string` or `undefined`. If it is a string,
we render the value and suit. Otherwise we render the "back" of the card - a
dark red rectangle. Cards have an optional `onClick()` event.

### Hand

A `HandView` renders several cards:

```typescript
type HandViewProps = {
    prefix: string;
    cards: (string | undefined)[];
    onClick?: (index: number) => void;
};

const HandView: React.FC<HandViewProps> = ({ cards, prefix, onClick }) => {
    return <div style={{ display: "flex", flexDirection: "row", justifyContent: "center" }}>{
            cards.map((card, i) => <CardView key={prefix + ":" + i} card={ card } onClick={() => { if (onClick) { onClick(i) } }} />)
        }
    </div>
}
```

This can be the player's hand, where we should have `string` values for each
card and an `onClick()` event hooked up for when the player clicks on a card to
discard it. It can also be the other player's hand, in which case we should have
`undefined` values for each card and just show their backs.

### Table

`MainView` implements a view of the whole table:

```typescript
const useSelector: TypedUseSelectorHook<RootState> = useReduxSelector;

const MainView = () => {
    const idSelector = useSelector((state) => state.id);
    const otherPlayer = useSelector((state) => state.otherPlayer);
    const gameStateSelector = useSelector((state) => state.gameStatus);
    const deckViewModel = useSelector((state) => state.deckViewModel);

    const myTurn = gameStateSelector.value === "MyTurn";

    const canDiscard = (index: number) => {
        if (deckViewModel.value.discardPile.length === 0) {
            return true;
        }

        return matchSuitOrValue(
            deckViewModel.value.myCards[index],
            deckViewModel.value.discardPile[deckViewModel.value.discardPile.length - 1]);
    }

    return <div>
        <div>
            <p>Id: {idSelector.value}</p>
            <p>Other player: {otherPlayer.value}</p>
            <p>Status: {gameStateSelector.value}</p>
        </div>
        <div style={{ height: 200, textAlign: "center" }}>
            <HandView prefix={"others"} cards={ new Array(deckViewModel.value.othersHand).fill(undefined) } />
        </div>
        <div style={{ height: 200, display: "flex", flexDirection: "row", justifyContent: "center" }}>
            <div style={{ display: deckViewModel.value.drawPile > 0 ? "block" : "none", margin: 5 }} onClick={() => { if (myTurn) { drawCard()} }}>
                <span>{deckViewModel.value.drawPile} card{deckViewModel.value.drawPile !== 1 ? "s" : ""}</span>
                <CardView card={ undefined } />
            </div>
            <div style={{ display: deckViewModel.value.discardPile.length > 0 ? "block" : "none", margin: 5 }}>
                <span>{deckViewModel.value.discardPile.length} card{deckViewModel.value.discardPile.length !== 1 ? "s" : ""}</span>
                <CardView card={ deckViewModel.value.discardPile[deckViewModel.value.discardPile.length - 1] } />
            </div>
        </div>
        <div style={{ height: 200, textAlign: "center" }}>
            <HandView
                prefix={"mine"}
                cards={ deckViewModel.value.myCards }
                onClick={(index) => { if (myTurn && canDiscard(index)) { discardCard(index) } }} />
        </div>
    </div>
```

This consists of:

* A top display showing our ID, the other player's ID, and the game status.
* The other player's hand (we'll only see the back of the cards).
* The draw pile - if there's no more cards in the draw pile, we don't show
  anything; otherwise we show the back of a card and the number of cards in the
  pile.
* The discard pile - if nothing discarded yet, we don't show anything; otherwise
  we show the card on top of the discard pile and the number of cards in the
  pile.
* Our hand.

If it is our turn, we hook up `drawCard()` to the draw pile's `onClick()` and
for each card we can discard, we hook up `discardCard()` to the card's
`onClick()`.

And that's it. Rendering it all on the page:

```typescript
const root = ReactDOM.createRoot(document.getElementById("root")!);
root.render(
    <Provider store={store}>
        <MainView />
    </Provider>
);
```

Here, `Provider` comes from the `react-redux` package and makes the Redux store
available to the React components.

## Initialization

Like with rock-paper-scissors, let's look at how we initialize the game:

```typescript
getLedger<Action>().then(async (ledger) => {
    const id = randomClientId();

    await store.dispatch(updateId(id));

    const queue = await upgradeTransport(2, id, ledger);

    await store.dispatch(updateQueue(queue));

    for (const action of ledger.getActions()) {
        if (action.clientId !== id) {
            await store.dispatch(updateOtherPlayer(action.clientId));
            break;
        }
    }

    const [sharedPrime, turnOrder] = await establishTurnOrder(2, id, queue);

    await store.dispatch(updateGameStatus("Shuffling"));

    const [keys, deck] = await shuffle(id, turnOrder, sharedPrime, getDeck(), queue, 64);
 
    const imFirst = turnOrder[0] === id;

    await store.dispatch(updateDeck(new Deck(deck, keys, store)));

    await deal(imFirst, 5);

    await store.dispatch(updateGameStatus(imFirst ? "MyTurn" : "OthersTurn"));

    if (!imFirst) {
        await waitForOpponent();
    }
});
```

* We connect to the Fluid session and get a reference to the `ledger`, as we saw
  in [part 7](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html).
* We generate a random client ID (using the implementation
  in [`packages/primitives/src/randomClientId.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/packages/primitives/src/randomClientId.ts)).
* Update our ID in the Redux store.
* We call `upgradeTransport()` (also discussed in [part 7](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html)).
* Update the Redux store with a reference to the async queue.
* We retrieve and store the other player’s ID.
* We get the shared prime and establish turn order (also covered in [part 7](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html)).
* We update the game status to `Shuffling`.
* We shuffle the deck using the `shuffle()` primitive and get back our keys and
  encrypted cards.
* Determine whether we are first (based on established turn order) and store
  this in `imFirst`.
* We instantiate a `Deck` and store in the Redux `store`.
* Deal 5 cards to each player using `deal()`.
* Update state again, based on whether we are first or not to `MyTurn` or
  `OthersTurn`.
* If we're not first to play, call `waitForOpponent()`.

This initialization is a bit longer than the one for rock-paper-scissors, since
we have to shuffle and deal cards, and the order in which the players go is
important.

## Summary

We looked at implementing a discard card game using the Mental Poker toolkit.
The full source code for the demo is under [`demos/discard`](https://github.com/vladris/mental-poker-toolkit/tree/main/demos/discard).

* Instructions on how to run the game in [`README.md`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/README.md).
* Deck management is implemented in [`deck.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/deck.ts).
* The rest of the model is implemented in [`model.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/model.ts).
* The Redux store is implemented in [`store.ts`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/store.ts).
* The React components are here:
  [`cardView.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/cardView.tsx),
  [`handView.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/handView.tsx),
  [`mainView.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/mainView.tsx).
* Initialization happens in [`index.tsx`](https://github.com/vladris/mental-poker-toolkit/blob/main/demos/discard/src/index.tsx).

We finally put the whole toolkit to its intended use and built an end-to-end interactive, 2-player card game.
