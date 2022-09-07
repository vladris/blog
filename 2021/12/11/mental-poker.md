# Mental Poker

For the past year or so, I've been on the [Fluid
Framework](https://fluidframework.com/) team. I won't go deeply into
the details of the framework, rather I'll quote a few paragraphs from
the [Overview page](https://fluidframework.com/docs/):

> **What is Fluid Framework?**
>
> Fluid Framework is a collection of client libraries for distributing
> and synchronizing shared state. These libraries allow multiple clients
> to simultaneously create and operate on shared data structures using
> coding patterns similar to those used to work with local data.
>
> **Why Fluid?**
>
> Because building low-latency, collaborative experiences is hard!
>
> Fluid Framework offers:
>
> * Client-centric application model with data persistence requiring
>   no custom server code.
> * Distributed data structures with familiar programming patterns.
> * Very low latency.
>
> Applications built with Fluid Framework require zero custom code on
> the server to enable sophisticated data sync scenarios such as
> real-time typing across text editors. Client developers can focus on
> customer experiences while letting Fluid do the work of keeping data
> in sync.
>
> **How Fluid works**
>
> Fluid was designed to deliver collaborative experiences with blazing
> performance. To achieve this goal, the team kept the server logic as
> simple and lightweight as possible. This approach helped ensure
> virtually instant syncing across clients with very low server costs.
>
> To keep the server simple, each Fluid client is responsible for its
> own state. While previous systems keep a source of truth on the
> server, the Fluid service is responsible for taking in data
> operations, sequencing the operations, and returning the sequenced
> operations to the clients. Each client is able to use that sequence to
> independently and accurately produce the current state regardless of
> the order it receives operations.
>
> The following is a typical flow.
>
> * Client code changes data locally.
> * Fluid runtime sends that change to the Fluid service.
> * Fluid service sequences that operation and broadcasts it to all
>   clients.
> * Fluid runtime incorporates that operation into local data and
>   raises a `valueChanged` event.
> * Client code handles that event (updates view, runs business
>   logic).

When using Fluid Framework, you model your data using a set of
[distributed data
structures](https://fluidframework.com/docs/build/dds/) which can
internally merge changes from multiple clients.

During various hackathons, the team built various applications using
this data model. Of course, one of the first applications of any new
technology is games. This got me thinking about how we could model a
game on top of the framework.

There are some interesting constraints: games like chess or go don't
have any hidden information, but most games do require some hidden
information. Card games are especially interesting: each player holds
some cards that only themselves can see, some cards are face up on the
table (everyone can see them), while the rest of the deck is face down
on the table (nobody sees what order the cards are in).

With Fluid Framework, data is replicated across all clients. Assuming
we're playing a game of high stakes poker, we can't trust any other
client not to cheat. So a naïve solution of sending the whole game state
(cards each player holds in their hand) to all clients and trust clients
not to peek won't work. We should assume that even if the game code
only shows a client their own cards, the client can cheat and use a
debugger to see what other players are holding in their hands.

We can trust the server, but there is very little the server can do for
us -while it can tell us which client changed state (distributed data
structure changes sequenced by the server include client ID), the server
itself cannot maintain private state. So, for example, we can't tell
the server to shuffle a deck of card without telling us what order the
cards end up in - all shared state is replicated across all clients.

In this zero-trust environment, where we assume other clients can cheat
and all shared state can be accessed by all clients, can we model a card
game? Surprisingly, the answer is *yes*.

## Mental Poker

Turns out this exact problem has been studied for quite some time,
starting with the original 1981
[paper](https://people.csail.mit.edu/rivest/pubs/SRA81.pdf) by Ron
Rivest, Adi Shamir, and Leonard Adleman (inventors of the RSA algorithm
among other things).

> *Once there were two mental chess experts who had become tired of
> their pastime. "Let's play mental poker, for variety" suggested
> one. "Sure" said the other. "Just let me deal!"*

Mental poker requires a commutative encryption function. If we encrypt
$A$ using $Key_1$ then encrypting the result using $Key_2$, we should be
able to decrypt the result back to $A$ regardless of the order of
decryption (first with $Key_1$ and then with $Key_2$, or vice-versa).

Here is how Alice and Bob play a game of mental poker:

* Alice takes a deck of cards (an array), shuffles the deck, generates
  a secret key $K_A$, and encrypts each card with $K_A$.
* Alice hands the shuffled and encrypted deck to Bob. At this point,
  Bob doesn't know what order the cards are in (since Alice encrypted
  the cards in the shuffled deck).
* Bob takes the deck, shuffles it, generates a secret key $K_B$, and
  encrypts each card with $K_B$.
* Bob hands the deck to Alice. At this point, neither Alice nor Bob
  know what order the cards are in. Alice got the deck back reshuffled
  and re-encrypted by Bob, so she no longer knows where each card
  ended up. Bob reshuffled an encrypted deck, so he also doesn't know
  where each card is.

At this point the cards are shuffled. In order to play, Alice and Bob
also need the capability to look at individual cards. In order to enable
this, the following steps must happen:

* Alice decrypts the shuffled deck with her secret key $K_A$. At this
  point she still doesn't know where each card is, as cards are still
  encrypted with $K_B$.
* Alice generates a new set of secret keys, one for each card in the
  deck. Assuming a 52-card deck, she generates
  $K_{A_1} ... K_{A_{52}}$ and encrypts each card in the deck with one
  of the keys.
* Alice hands the deck of cards to Bob. At this point, each card is
  encrypted by Bob's key, $B_K$, and one of Alice's keys, $K_{A_i}$.
* Bob decrypts the cards using his key $K_B$. He still doesn't know
  where each card is, as now the cards are encrypted with Alice's
  keys.
* Bob generates another set of secret keys, $K_{B_1} ... K_{B_{52}}$,
  and encrypts each card in the deck.
* Now each card in the deck is encrypted with a unique key that only
  Alice knows and a unique key only Bob knows.

If Alice wants to look at a card, she asks Bob for his key for that
card. For example, if Alice draws the first card, encrypted with
$K_{A_1}$ and $K_{B_1}$, she asks Bob for $K_{B_1}$. If Bob sends her
$K_{B_1}$, she now has both keys to decrypt the card and "look" at it.
Bob still can't decrypt it because he doesn't have $K_{A_1}$.

This way, as long as both Alice and Bob agree that one of them is
supposed to "see" a card, they exchange keys as needed to enable this.

At the end of the game, players reveal all keys to validate that no
cheating happened.

This approach can be extended to any number of players, each player
maintaining their own set of secret keys.

## Modeling a Game

We can model a game using two data structures: one to keep track of the
cards, one to keep track of the "moves" in the game.

We can model a deck of cards using a distributed data structure that
holds the set of cards. Each client generates secret keys and initially
keeps them private (not part of the shared state). The deck of cards can
be shuffled and encrypted as described above, with each client updating
the shared set of cards.

We can model the gameplay using an append-only list of moves. For
example, if Alice "draws" the first card, the move can be modeled as
`DRAW 1`. If Bob agrees Alice should see the card, Bob can publish his
secret key $K_{B_1}$ as `PUBLISH <KB1>`. Alice can now use her
$K_{A_1}$ and the published $K_{B_1}$ to decrypt the first card of the
deck (stored in the other data structure). `DRAW`, `PUBLISH`, and other
actions are part of the game semantics, which can be implemented and
interpreted by clients.

Note the deck of cards stays in place during the game. "Drawing" a
card means simply that all clients agree Alice should get the keys to
the card at index 1 and that the next card to be drawn is at index 2.
"Discarding" a card simply means Bob said he discards the card at
index 5. Depending on whether discarding is face up or face down, Bob
can publish $K_{B_5}$ or keep it private until the end of the game. All
these actions are part of the game move list, and clients can construct
the game state based on these, without having to mutate the deck itself.

In terms of trust, we can say that, at any point, if a client can prove
the game is invalid (another client misbehaved), the game is cancelled.
If a player acts out of turn, or performs an action that they
shouldn't, the game is invalid. At the end of the game, the append-only
list should contain the full record of moves. With all keys available,
clients can replay and validate no cheating happened (for example Bob
claiming a card decrypted to an Ace, when in fact the card was a 2).
Clients can keep a local copy of the list of moves, and confirm no other
client rewrote history by tweaking the content of the list.

Establishing turn order can also be modeled through the append-only
action list: each player can start by adding a `SIT AT TABLE` action.
The framework will sequence these action in some order, which will
become the turn order. For example, if both Alice and Bob concurrently
`SIT AT TABLE`, the action list will contain both actions in some order.
Alice and Bob will take turns in that order.

Game semantics can be implemented as actions clients interpret. This is
outside the scope of this article.

## Resources

As I mentioned, this problem has been studied for many decades. [A
Toolbox for Mental Card
Games](https://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=589CF11E796235A23376DFE85C69B96E?doi=10.1.1.29.6679&rep=rep1&type=pdf)
by Christian Schindelhauer describes many other techniques for playing
cards in a zero-trust environment.

There is also an open-source C++ library implementing the toolbox:
[LibTMCG](http://www.nongnu.org/libtmcg/).

The <https://secret.cards> website seems to implement a card game using
mental poker techniques.

Wikipedia also has a good page on [mental
poker](https://en.wikipedia.org/wiki/Mental_poker).
