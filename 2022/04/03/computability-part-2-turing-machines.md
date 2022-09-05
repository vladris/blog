# Computability Part 2: Turing Machines

In the [previous
post](https://vladris.com/blog/2022/02/12/computability-part-1-a-short-history.html),
we looked at a history of what would become computer science. In this
post, we'll focus on Turing machines and Turing completeness.

The informal definition we gave to a Turing machine in the previous post
is:

> An abstract computer consisting of an infinite tape of cells, a head
> that can read from a cell, write to a cell, and move left or right
> over the tape, and a set of rules which direct the head based on the
> read symbol and the current state of the machine.

Formally:

> A Turing machine is a 7-tuple
> $M = \langle Q, q_0, F, \Gamma, b, \Sigma, \delta \rangle$.
>
> * $Q \ne \varnothing$ is a finite set of *states*. These are all the
>   states the machine can be in.
> * $q_0 \in Q$ is the *initial state*. This is the state the machine
>   starts in.
> * $F \subseteq Q$ is the set of *final states*. When the machine
>   reaches one of the final states, it *halts* - it stops execution.
> * $\Gamma \ne \varnothing$ is a finite set of *tape symbols*. These
>   are all the symbols that can appear on the tape.
> * $b \in \Gamma$ is the *blank symbol*, one of the possible *tape
>   symbols*. The only symbol allowed to occur on the tape infinitely
>   often at any step.
> * $\Sigma \subseteq \Gamma \setminus \lbrace b \rbrace$ is the set
>   of *input symbols* allowed to appear in the initial tape contents
>   (not written by the machine during execution). These symbols can
>   be the whole alphabet (except the blank symbol), or a subset of
>   the alphabet.
> * $\delta: (Q \setminus F) \times \Gamma \to Q \times \Gamma \times \lbrace L, R \rbrace$
>   is a function called the *transition function*. This functions
>   takes as input the current machine state and the symbol on the
>   tape. It outputs the new machine state, the symbol to overwrite
>   the current tape symbol, and the head movement (either left or
>   right). Note the function domain excludes the final states - once
>   the machine reaches a state in $F$, it halts so no more
>   transitions happen.

Alternately, the transition function can be defined as a partial
function
$\delta: Q \times \Gamma \hookrightarrow Q \times \Gamma \times \lbrace L, R \rbrace$,
where the machine halts if the function is undefined for the given
combination of machine state and tape symbol. In some compact Turing
machines (like we'll see below), $F$ is empty. There is not *final
state*, rather we halt when encountering a certain combination of
machine state and tape symbol for which no transition is defined.

Note this definition allows for some very uninteresting machines: a
machine that only has an initial and a final state
($Q = \lbrace q_0, f \rbrace$) and, for any input symbol in $\Gamma$,
the transition function moves the machine into the final state. This is
a Turing machine, but it can't really compute much. Something more is
needed.

## Universal Turing machines

A *universal Turing machine* is a Turing machine that can simulate
another, arbitrary, Turing machine on arbitrary input. That is, it can
read the description of a Turing machine and that machine's input as
its own input, then simulate the execution of that machine.

With this definition, a universal Turing machine can compute anything
any other Turing machine can compute (anything that is computable).

Marvin Minsky discovered a universal Turing machine that requires only 7
states and 2 symbols. Yurii Rogozhin discovered a machine with only 4
states and 6 symbols. Let's call the states
$Q = \lbrace A, B, C, D \rbrace$ and the symbols
$\Gamma = \lbrace 0, 1, 2, 3, 4, 5 \rbrace$.

**(4, 6) Turing Machine**

|   |   A   |   B   |   C   |   D   |
|:-:|:-----:|:-----:|:-----:|:-----:|  
| 0 | 3,L,A | 4,R,B | 0,R,C | 4,R,D |
| 1 | 2,R,A | 2,L,C | 3,R,D | 5,L,B |
| 2 | 1,L,A | 3,R,B | 1,R,C | 3,R,D |
| 3 | 4,R,A | 2,L,B | HALT  | HALT  |
| 4 | 3,L,A | 0,L,B | 5,R,A | 5,L,B |
| 5 | 4,R,D | 1,R,B | 0,R,A | 1,R,D |

The above table describes the transition function of the Turing machine.
For example, if the machine is in state `A` and the read tape symbol is
`5`, we can look up the `A` column and `5` row to find the transition
`4,R,D`. This means "print `4` on the tape (overwriting the current
symbol), move the head right (`R`), machine is now in state `D`".

We're using the partial transition function definition, so instead of
defining one or more explicit final states ($F$), we don't define a
transition when the tape symbol is `3` and the machine is in state `C`
or state `D`.

## Implementation

Let's look at a Python implementation of Turing machines. First, let's
implement the tape we will be using. Theoretically this is an infinite
tape. To simulate this in software, we will use a list and whenever we
move the head left or right beyond the list, we extend the list with an
additional blank symbol:

``` python
class Tape:
    def __init__(self, tape, head = 0):
        # Initial tape should have at least one symbol
        assert(len(tape) >= 1)
        # Tape head should be a valid index
        assert(0 <= head < len(tape))

        self.tape = tape
        self.head = head

    def read(self):
        return self.tape[self.head]

    def write(self, symbol):
        self.tape[self.head] = symbol

    def move_left(self):
        # If attempting to move left out of bounds, extend tape left
        if self.head == 0:
            self.tape.insert(0, 0)
        else:
            self.head -= 1

    def move_right(self):
        self.head += 1
        # If attempting to move right out of bounds, extend tape right
        if self.head == len(self.tape):
            self.tape.append(0)
```

We'll implement a machine that takes a tape, a transition table, and an
initial state, and runs until it halts:

``` python
def machine(tape, transitions, state):
    while True:
        symbol = tape.read()

        # If no transition is defined for the current state and symbol, halt
        if not transitions[state][symbol]:
            break

        new_symbol, direction, new_state = transitions[state][symbol]

        tape.write(new_symbol)
        tape.move_left() if direction == 'L' else tape.move_right()
        state = new_state
```

To stich this together, we need a transition table and initial tape
state. We'll use the Rogozhin (4, 6) machine:

``` python
## Machine states
A, B, C, D = 'A', 'B', 'C', 'D'

## Left and right
L, R = 'L', 'R'

## Rogozhin 4-state, 6-symbol Turing machine
transition = {
    A: [(3, L, A), (2, R, A), (1, L, A), (4, R, A), (3, L, A), (4, R, D)],
    B: [(4, R, B), (2, L, C), (3, R, B), (2, L, B), (0, L, B), (1, R, B)],
    C: [(0, R, C), (3, R, D), (1, R, C), None, (5, R, A), (0, R, A)],
    D: [(4, R, D), (5, L, B), (3, R, D), None, (5, L, B), (1, R, D)],
}
```

This machine is a universal Turing machine, meaning it can simulate any
other turing machine, thus is capable of universal computation (can
compute anything that is computable).

## Turing-completeness

> A Turing-complete system is any system capable of simulating any
> Turing machine.

Turing-completeness is a way of expressing the computational power of a
given system. A Turing-complete system is capable of universal
computation. The small Rogozhin (4, 6) machine, since it is a universal
Turing machine, is Turing-complete.

More so, the fact that we can simulate this machine in the Python
programming language proves that the Python language itself is
Turing-complete.

## Esoteric Turing-complete systems

If we weaken some of the constraints for Turing machines, there are even
smaller *weak* universal Turing machines. For example, if we allow the
tape to contain an infinitely repeated sequence of symbols, or we don't
require the machine to ever halt.

The smallest weak Turing machine is a Turing machine consisting of 2
states and 3 symbols. Let's call the states $Q = \lbrace A, B \rbrace$
and the symbols $\Gamma = \lbrace 0, 1, 2 \rbrace$.

**(2, 3) Turing Machine**

|   |   A   |   B   |
|:-:|:-----:|:-----:|
| 0 | 1,R,B | 2,L,A |
| 1 | 2,L,A | 2,R,B |
| 2 | 1,L,A | 0,R,A |

Stephen Wolfram in [A New Kind of
Science](https://www.goodreads.com/book/show/238558.A_New_Kind_of_Science)
(a book we'll get back to in a future post) described a 2-state
5-symbol universal Turing machine and conjectured the 2-state 3-symbol
machine is also universal. The universality of the 2-state 3-symbol
machine was proved in 2007.

In terms of Turing-complete programming languages, a somewhat famous
esoteric programming langue is
[Brainfuck](https://en.wikipedia.org/wiki/Brainfuck). Brainfuck uses a
byte array (tape), a data pointer (index in the array), and 8 symbols:
`>`, `<`, `+`, `-`, `.`, `,`, `[`, `]`. The symbols are interpreted as:

* `>`: Increment the data pointer (move head right).
* `<`: Decrement the data pointer (move head left).
* `+`: Increment array value at data pointer.
* `-`: Decrement array value at data pointer.
* `.`: Output value at data pointer.
* `,`: Read 1 byte of input and store at data pointer.
* `[`: If the byte at data pointer is 0, jump right to the matching
  `]`, else increment data pointer
* `]`: If the byte at data pointer is not 0, jump left to the matching
  `[`, else decrement data pointer

This simple language is very much modeled after a Turing machine. Here
is "Hell World!" in Brainfuck:

``` text
++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.>
---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
```

Since the language definition is so simple, it is very easy to write a
Brainfuck interpreter:

``` python
import sys

def bf(program):
    # Data array, data pointer, and code pointer
    data, dp, cp = [0], 0, 0

    while cp < len(program):
        match program[cp]:
            case '<':
                dp -= 1
            case '>':
                dp += 1
                if dp == len(data):
                    data.append(0)
            case '+':
                data[dp] += 1
            case '-':
                data[dp] -= 1
            case '.':
                print(chr(data[dp]), end='')
            case ',':
                data[dp] = ord(sys.stdin.read(1))
            case '[':
                if data[dp] == 0:
                    opened = 1
                    while opened:
                        cp += 1
                        if program[cp] == ']':
                            opened -= 1
                        elif program[cp] == '[':
                            opened += 1
            case ']':
                if data[dp] != 0:
                    opened = 1
                    while opened:
                        cp -= 1
                        if program[cp] == '[':
                            opened -= 1
                        elif program[cp] == ']':
                            opened += 1
        cp += 1
```

Also note that any programming language that can implement a Brainfuck
interpreter is Turing-complete (since Brainfuck is Turing-complete).

There's also some surprising proofs of unintentional
Turing-completeness. For example, C++ template metaprogramming was
proved to be Turing-complete (not the C++ language itself, which is
obviously Turing-complete, just the template part alone). Magic: The
Gathering [is also Turing-complete](https://arxiv.org/abs/1904.09828).
Turing-completeness comes in many forms. In the next posts, we'll look
at some other models of universal computation: tag systems and cellular
automata.
