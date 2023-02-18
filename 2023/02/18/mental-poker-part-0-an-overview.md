# Mental Poker Part 0: An Overview

I wrote previously about [Mental Poker](https://vladris.com/blog/2021/12/11/mental-poker.html),
how one can set up a game in a zero trust environment, and how this could be
implemented using [Fluid Framework](https://fluidframework.com/).

Since the previous post, I spent some more time prototyping an implementation
with a colleague and did a [tech talk](https://www.youtube.com/watch?v=F1gPTXAllxY)
about it.

If you haven't read the previous post and are not familiar with Mental Poker,
the following won't make much sense. Please start there or by watching the
tech talk video.

The implementation consists of a few components:

* A Fluid Framework append-only list distributed data structure - Used for
  tracking turns in a game.
* Cryptography - An implementation of SRA (commutative encryption algorithm
  required by Mental Poker) and digital signing (required for authenticating
  messages from players).
* Game client - A layer that abstracts communication between clients, with
  an implementation on Fluid Framework.
* A state machine - Used to model games (**If I make this move I expect the
  other player to make that move**).
* Recipes built on top of the state machine, like shuffling a deck of cards
  using the steps described in my [previous blog post](https://vladris.com/blog/2021/12/11/mental-poker.html).

At the time of writing, the append-only list distributed data structure is
ready, available on my GitHub as [fluid-ledge](https://github.com/vladris/fluid-ledger)
and published on [npm](https://www.npmjs.com/package/fluid-ledger-dds).

The other components will all eventually end up in the
[mental-poker-toolkit](https://github.com/vladris/mental-poker-toolkit) repo.

Some parts, like cryptography and the game client, I cleaned up and moved from
a private hackathon repo. Other parts, like the state machine, require major
rework, which I haven't gotten around to yet.

The plan is to provide a quality implementation with good documentation and
samples. A major difference between the hackathon proof of concept and this is
that the proof of concept implements a simple discard game while I'm hoping the
toolkit can support games with more than two players.

## Discard game

Modeling a game like Poker is non-trivial. That said, a big part of the
complexity comes from the rules of the game itself. For a proof of concept of
Mental Poker, we didn't want to get in the weeds of Poker rules, rather
showcase the key ideas of how two players can shuffle a deck of cards, agree
on what order the cards end up in, while at the same time each being able
to maintain some private state (cards in hand). All of this done over a public
channel (Fluid Framework).

The game we modeled was simple: players draw a hand of cards, then take turns
discarding by number or suit. If a player can't discard (no matching number or
suit), they draw cards until they can discard. The player who discards their
whole hand first wins.

This prototype informed the components we had to build.

## A new distributed data structure

Framework does not offer "out of the box" a data structure like the one needed
to model a sequence of moves. We ended up using `SharedObjectSequence`, a data
structure that was marked as deprecated and since removed from Fluid. In
general, the Fluid data structures that support lists are overkill for Mental
Poker as they support insertion and deletion of sequences of items at arbitrary
positions. For modeling a game, we just need an append only list - players take
turns and each *move* means appending something to the end of the list.

In fact, having an append-only list ensures that we don't run into issues like
a client unexpectedly inserting something in the middle of the list, which
doesn't make sense if we're modeling a sequence of moves in a game.

## Cryptography

I was also not able to find a package providing commutative encryption. This is
a key requirement for the Mental Poker protocol but industry standard
cryptography algorithms do not have this property. I ended up implementing the
SRA algorithm from scratch, including a bunch of BigInt math. I still strongly
believe in the *don't roll your own crypto* rule, so please do not use my
implementation to play Poker for real money.

Besides encryption, we also need digital signatures. When a player joins a
game, they generate a public/private key pair and their first action is to post
their public key. All subsequent moves from that player are signed with the
private key, so other players can ensure the action is taken by the player
claiming to take that action, eliminating spoofing. Fortunately we were able to
use `Crypto.subtle` for this (see [Crypto Web API](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/subtle)).

## State machine

Another interesting discovery was the state machine. A high-level game move,
like *I'm drawing a card from the top of the pile* translates into a message
exchange between the players:

* `Alice`: I'm drawing a card from the top of the pile.
* `Bob`: Here is my key for that card.

Shuffling cards, as described in the previous blog post, includes a longer
sequence of steps. We needed a way to express "I do *this*, then I expect the
other player to reply with *that*". We can use such a state machine to express
sequences of multiple moves to implement things like card shuffling.

The proof of concept state machine uses a queue of expected moves from the
other player to implement the game mechanics and Mental Poker protocol. For
example, for the Discard game, if it is the other player's turn, we expect
two things can happen: they either discard a card or draw a card.

If they discard a card, then they publish their encryption key for the card
which we can use to *see* the card (again, please refer to the previous
[Mental Poker post](https://vladris.com/blog/2021/12/11/mental-poker.html) for
details on the protocol). Alternately, if they can't discard a card, they
need to draw a card, in which case we have to hand over our encryption key
for the card on top of the deck.

## Recipes

Some of the rules captured in this state machine are specific to each game
implemented. Others though are simply steps in the Mental Poker protocol:
things like shuffling, drawing cards etc. are all modeled as *actions I take
and actions I expect the other player to follow up with*. I envision
expressing such known sequences as "recipes", building blocks for games.

As I mentioned before, the proof of concept state machine implementation
requires some major rework. It needs to scale from two players to an arbitrary
number of players, and needs to support recipes, which it currently doesn't.
At the time of writing, this is one of the biggest chunks of pending work, and
considering this is a hobby project I work on when time permits, I currently
don't have a good sense of when I'll finish this. That said, a bunch of pieces
are already in decent shape and public, so I plan to write about them while
I continue working on finishing the toolkit.

## Mental Poker series

In upcoming blog posts, I plan to cover the various pieces discussed above. The
components address different problems, and I find all of them quite interesting.
The problem space includes understanding how Fluid Framework distributed data
structures work internally, how to generate large prime numbers, and how to
model expected sequences of moves in a game among other things.

This post outlines the high level framing of the project. Following posts will
dive deep into specific aspects.

In terms of applications, as I mention in the tech talk, the term *games* is
pretty broad - we're not talking only about card games, but things like
auctions, lotteries, blind voting etc. All of these can be implemented using
Mental Poker as decentralized, zero-trust games.
