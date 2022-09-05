# Computability Part 4: Conway's Game of Life

<script src="https://vladris.com/static/js/gol.js"></script>

In the [previous
post](https://vladris.com/blog/2022/05/20/computability-part-3-tag-systems.html)
we talked about tag systems. In this post (and the next one) we will
cover cellular automata. The most famous cellular automata is Conway's
Game of Life. Before providing the formal definitions, here is the Game
of Life in action:

<div id="demo" style="overflow: scroll; text-align: center"></div>
<script>animate(50, 30, [[14, 25], [15, 24], [15, 25], [15, 26], [16, 24]], "demo");</script>

Formal definition:

> A cellular automaton consists of a discrete n-dimensional lattice of
> cells, a set of states (for each cell), a notion of neighborhood for
> each cell, and a transition function mapping the neighborhood of each
> cell to a new cell state.

The system evolves over time, where at each step, the transformation
function is applied over the lattice to determine the states of the next
generation of cells.

Conway's Game of Life is a cellular automaton on a 2D plane with the
following rules:

> 1. Any live cell with fewer than two live neighbors dies.
> 2. Any live cell with two or three live neighbors lives on to the
>    next generation.
> 3. Any live cell with more than three live neighbors dies.
> 4. Any dead cell with exactly three live neighbors becomes a live
>    cell.

In other words, a live cell stays alive during the next iteration if it
has 2 or 3 live neighbors. A dead cell becomes live if it has exactly 3
live neighbors.

In the case of Conway's Game of Life, the lattice is a 2D grid, we have
2 states (*on* or *off*), the neighborhood of a cell consists of all
adjacent cells (including corners), and the transition function is the
one described above. Mathematician John Conway proposed the Game of Life
in 1970.

The reason we started with Conway's Game of Life for discussing
cellular automata is that this simple game with simple rules exhibits
some very interesting behavior that has been classified for many years
by people toying with the simulation.

First, we have *still lives*, patterns that don't change while stepping
through the simulation. These patterns are stable: no cells die, no
cells become live.

<div id="still" style="overflow: scroll; text-align: center"></div>
<script>animate(50, 30, [[9, 14], [9, 15], [10, 14], [10, 15], [10, 33], [9, 34], [9, 35], [11, 34], [11, 35], [10, 36], [19, 14], [20, 13], [20, 15], [21, 14], [19, 33], [19, 34], [20, 33], [20, 35], [21, 34]], "still");</script>

Next, we have *oscillators*, patterns that repeat with a certain
periodicity:

<div id="oscillator" style="overflow: scroll; text-align: center"></div>
<script>animate(50, 30, [[9, 14], [9, 15], [9, 16], [10, 33], [10, 34], [10, 35], [9, 34], [9, 35], [9, 36], [19, 14], [20, 14], [19, 15], [21, 17], [22, 16], [22, 17], [19, 31], [20, 31], [18, 32], [21, 32], [17, 33], [22, 33], [16, 34], [23, 34], [16, 35], [23, 35], [17, 36], [22, 36], [18, 37], [21, 37], [19, 38], [20, 38]], "oscillator");</script>

In the above example, the last (bottom right) pattern has period 5 and
is called *Octagon 2*. The other 3 patterns all have period 2.

More interestingly, we have *spaceships* - these are patterns that
repeat but translate through space:

<div id="ships1" style="overflow: scroll; text-align: center"></div>
<script>animate(30, 30, [[14, 14], [15, 15], [16, 13], [16, 14], [16, 15]], "ships1");</script>

<div id="ships2" style="overflow: scroll; text-align: center"></div>
<script>animate(30, 30, [[14, 14], [15, 14], [16, 14], [13, 15], [16, 15], [16, 16], [16, 17], [13, 18], [15, 18]], "ships2");</script>

The above examples shows a couple of small spaceships, the tiny 5-cell
*glider* and the *lightweight spaceship* or *LWSS*. There are many more
spaceship patterns, some of them quite large (hundreds or even thousands
of cells).

Most simulations tend to eventually stabilize into a combination of
oscillators and still lives. Patterns that start from a small seed of a
handful of cells and take a long time (in terms of iterations) to
stabilize are called *Methuselahs*. Here is an example, nicknamed
*Acorn*:

<div id="acorn" style="overflow: scroll; text-align: center"></div>
<script>animate(50, 30, [[13, 22], [15, 21], [15, 22], [14, 24], [15, 25], [15, 26], [15, 27]], "acorn");</script>

Conway conjectured that for any initial configuration, there is an upper
limit of how many live cells can ever exist. This was proved wrong by
the discovery of *glider guns*. A glider gun generates gliders every few
iterations. The gliders continue moving away from the gun, thus running
the simulation the number of live cells continues to grow.

One of the most popular glider guns is called *Gosper glider gun*, named
after Mathematician and programmer Bill Gosper:

<div id="gun" style="overflow: scroll; text-align: center"></div>
<script>animate(50, 30, [[5, 1], [6, 1], [5, 2], [6, 2], [5, 11], [6, 11], [7, 11], [4, 12], [8, 12], [3, 13], [9, 13], [3, 14], [9, 14], [6, 15], [4, 16], [8, 16], [5, 17], [6, 17], [7, 17], [6, 18], [3, 21], [4, 21], [5, 21], [3, 22], [4, 22], [5, 22], [2, 23], [6, 23], [1, 25], [2, 25], [6, 25], [7, 25], [3, 35], [4, 35], [3, 36], [4, 36]], "gun", false);</script>

There are many other interesting patterns and constructions in the Game
of Life discovered throughout the years. A few examples:

* *Eaters* are still life or oscillator patterns that can interact
  and, over a number of iterations, "absorb" other patterns like
  spaceships, and return to their original state.
* *Reflectors* are still life or oscillator patterns that can change
  the direction of incoming spaceships, and return to their original
  state.
* *Puffers* are patterns that move like spaceships but leave behind a
  trail of patterns in their wake (unlike spaceships that cleanly
  translate).

There are many others, and combinations of them which give rise to
interesting systems like circuits and logic gates based on spaceships
and strategically placed still lives and oscillators.

## Implementation

Let's look at a Python implementation for the Game of Life. We will use
a wrap-around space, so we'll consider cells on the last column to be
neighbors with cells on the first column and similarly cells on the last
row to be neighbors with cells on the first row.

``` python
def make_matrix(width, height):
    return [[False] * width for _ in range(height)]

def neighbors(m, i, j):
    last_j = j + 1 if j + 1 < len(m[0]) else 0
    last_i = i + 1 if i + 1 < len(m) else 0

    return (m[i - 1][j - 1] + m[i - 1][j] + m[i - 1][last_j] +
        m[i][j - 1] + m[i][last_j] +
        m[last_i][j - 1] + m[last_i][j] + m[last_i][last_j])

def step(m1):
    m2 = make_matrix(len(m1[0]), len(m1))

    for i in range(len(m1)):
        for j in range(len(m1[0])):
            n = neighbors(m1, i, j)
            if n == 3:
                m2[i][j] = True
            elif n == 2 and m1[i][j]:
                m2[i][j] = True
    return m2
```

To run a simulation, we also need a function to print the game state and
some initial conditions:

``` python
def print_matrix(m):
    for line in m:
        print(str.join('', ['#' if c else ' ' for c in line]))

m = make_matrix(10, 10)

m[0][1] = True
m[1][2] = True
m[2][0] = True
m[2][1] = True
m[2][2] = True

for _ in range(100):
    print_matrix(m)
    m = step(m)
```

Another very simple to implement system with powerful computational
capabilities.

## Turing completeness

It turns out the Game of Life is Turing complete, meaning it is also
capable of universal computation. Gliders are key to this. In general,
if the behavior of cells would be either repetitive (still life or
oscillators cycle through 1 or more patterns) or chaotic, it would be
hard to perform any computation. But gliders "move" and can interact
with each other, thus enabling some non-chaotic processes.

We briefly discussed above how Game of Life patterns can be combined to
form circuits that can process signals (in the form of spaceships) like
logic gates and "memory storage". Paul Rendell "implemented" a
universal Turing machine in the Game of Life. His website
(<http://rendell-attic.org/gol/tm.htm>) covers the details, which we
won't go into due to the complexity. Suffice to say the patterns
emerging in the Game of Life can be combined to build such a device.
Paul also wrote a book about it[^1].

We again encountered a system capable of computing anything computable,
based only on a matrix of cells and a couple of rules (live cells with 2
or 3 neighbors stay alive, dead cells with exactly 3 neighbors become
live).

The website <https://conwaylife.com/> includes a lot of details on
Conway's Game of Life, various patterns discovered, and a forum where
people discuss their exploration of the system.

In the next post, we'll look at even simpler cellular automata:
elementary cellular automata where cells have 2 possible states and 2
neighbors.

[^1]: See [Turing Machine Universality of the Game of
    Life](https://www.amazon.com/Machine-Universality-Emergence-Complexity-Computation-ebook/dp/B012A45DVO/).
