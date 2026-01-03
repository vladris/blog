# Notes on Advent of Code 2025

These are my notes on the 2025 [Advent of Code](https://adventofcode.com/).

All my solutions are on my GitHub [here](https://github.com/vladris/aoc). And
here is the disclaimer I've been using with each post:

> **Disclaimer on my solutions**
>
> I use Python because I find it easiest for this type of coding. I treat
solving these as a write-only exercise. I do it for the problem-solving bit, so
I don't comment the code & once I find the solution I consider it done - I don’t
revisit and try to optimize even though sometimes I strongly feel like there is
a better solution. I don't even share code between part 1 and part 2 - once part
1 is solved, I copy/paste the solution and change it to solve part 2, so each
can be run independently. I also rarely use libraries, and when I do it's some
standard ones like `re`, `itertools`, or `math`. The code has no comments and is
littered with magic numbers and strange variable names. This is not how I
usually code, rather my decadent holiday indulgence. I wasn't thinking I will
end up writing a blog post discussing my solutions so I would like to apologize
for the code being hard to read.

Some thoughts first: this year's Advent of Code was shorter - only 12 days
instead of the usual 25 days. I also have mixed feelings about a couple of the
problems. Day 10 made me break my rule of not using external libraries. Day 12
was more of a prank. I'll get into details below.

## Day 1: Secret Entrance

Problem statement is [here](https://adventofcode.com/2025/day/1).

This was a very easy problem, not worth covering.

## Day 2: Gift Shop

Problem statement is [here](https://adventofcode.com/2025/day/2).

This one had potential, I was disappointed you can just count through each
number in a range and check if it's invalid.

I'm pretty sure there's a very efficient way to do it even for very large
ranges. E.g. between 12340000 and 12349999 there's a single invalid 12341234.
There must be a formula that yields the count of invalid numbers for an
arbitrary range just by doing the math on the range bounds + account for corner
cases. But I didn't spend the time on it since I could just check every single
number in the range.

## Day 3: Lobby

Problem statement is [here](https://adventofcode.com/2025/day/3).

Another easy one - for part 1, I simply checked all combinations of 2 digits
and picked the maximum.

For part 2, looking at all combinations doesn't work as we need to pick 12
digits out of each number. Luckily, a greedy approach works. I split the number
into a head (initially empty), and tail (all digits) and we repeat the following
algorithm 12 times: pick the largest digits from the tail that still leaves
enough digits in the tail for us to pick 12 digits total, add it to the head,
and cut the tail at the selected digit.

In other words, we are guaranteed to form the largest number if we keep picking
the largest available highest-order digit.

## Day 4: Printing Department

Problem statement is [here](https://adventofcode.com/2025/day/4).

Part 1 is trivial.

Part 2 is pretty easy too - the only "trick" is to not update the grid in-place
when removing rolls. Like a game of life simulation.

## Day 5: Cafeteria

Problem statement is [here](https://adventofcode.com/2025/day/5).

Part 1 is again trivial.

For part 2, we want to merge overlapping ranges first, then just count the
length of each range to arrive at the solution.

## Day 6: Trash Compactor

Problem statement is [here](https://adventofcode.com/2025/day/6).

In part 1, based on whether the operation is addition or multiplication, we
start with `0` or `1` then simply apply the operation (`+` or `*`) to all the
numbers in a column.

In part 2, we also go column by column, but we do it by digit rather than by
number. The right-to-left arrangement is a red herring - both addition and
multiplication are commutative so it doesn't matter whether we start from the
left or from the right.

## Day 7: Laboratories

Problem statement is [here](https://adventofcode.com/2025/day/7).

Part 1 - we just go down row by row from the top, counting hits and splitting
the beam when we encounter a `^`:

```python
hits = 0

for line in lines:
    new_beams = set()
    for beam in beams:
        if line[beam] == "^":
            hits += 1
            new_beams.add(beam - 1)
            new_beams.add(beam + 1)
        else:
            new_beams.add(beam)
    beams = new_beams
```

Part 2 is similar, the only addition being we need to keep a running tally of
how many beams (from different timelines) are at a given position.

```python
beams = {lines[0].index("S"): 1}

for line in lines: 
    new_beams = defaultdict(int)
    for beam in beams:
        if line[beam] == "^":
            new_beams[beam - 1] += beams[beam]
            new_beams[beam + 1] += beams[beam]
        else:
            new_beams[beam] += beams[beam]
    beams = new_beams
```

## Day 8: Playground

Problem statement is [here](https://adventofcode.com/2025/day/8).

This was a nice graph problem. For part 1, we can compute the distance
between each pair of nodes (junction boxes), sort by distance, pick the smallest
1000 pairs and add edges between them. Now we have our final graph. The next
step is to identify all the connected components of the graph and pick the three
with the largest number of nodes.

For part 2, note we can't just add edges to all except the last pair of nodes in
the solution for part 1, as those two nodes might already be part of the same
component. My solution here was to also precompute the distances between nodes
then, as I add edges between them, keep track of how many connected components
we have. Instead of adding 1000 edges, we keep connecting pairs of nodes until
we are left with two connected components.

## Day 9: Movie Theater

Problem statement is [here](https://adventofcode.com/2025/day/9).

Part 1 was easy, as we can just compute the area for every pair of points and
pick the largest one.

I found Part 2 very nice. I'm not sure I have the optimal solution, but here's
how I solved it: My initial idea was to "draw" lines between the points, then
"flood fill" the area they describe. Then we can check whether a pair of points
describes an area that is fully within the filled surface or not, in which case
it is invalid. If we do this, we can pick the largest valid area (fully within)
the filled surface.

The only problem with this approach is that the distances between points are
very large, so going point by point to connect the surface and fill it gets too
expensive. My solution was to scale down the grid.

Here's an example based on the smaller-sized sample input. The sample input is:

```text
..............
.......#...#..
..............
..#....#......
..............
..#......#....
..............
.........#.#..
..............
```

We can scale this down to:

```text
..#...#
.......
#.#....
.......
#...#..
.......
....#.#
```

Here's how we do it: we take the set of `X` coordinates of all points, sort it
then map $x_0 \rightarrow 0$, $x_1 \rightarrow 2$, $x_2 \rightarrow 4$ etc. We
do the same for the `Y` coordinates.

Now connecting the points and filling the surface becomes much more tractable.
We can simply update the grid with `#`s to connect a pair of points and we can
similarly flood-fill with `#` since the surface area is significantly scaled
down. We're no longer dealing with tens of thousands of points on the grid
between two points we have to connect.

Once we have this, we can again iterate over each pair of points and see if the
area they describe contains any `.` (which would make the area invalid), or not.
To determine which of the valid areas is the largest, we simply need to map back
the $(x_i, y_i)$ and $(x_j, y_j)$ from our scaled down grid to the actual
coordinate values. This will give us the "real" area size and by this point we
are guaranteed the area is valid (no `.` within its bounds).

## Day 10: Factory

Problem statement is [here](https://adventofcode.com/2025/day/10).

For part 1, note that pushing a button toggles a set of lights. With lights
being either "on" or "off", the toggle is an `XOR` applied to these lights.
For each available button, we either press it or we don't. Applying `XOR`
twice simply undoes its effect. The order in which we press buttons also doesn't
matter, since `XOR` is commutative. Solving part 1, for each button, we try
pressing it and not pressing it. We keep track of the number minimum number of
presses when we reach the desired end state, which is our solution.

I represented the lights as a bitmask, so `[.##.]` becomes `0b0110`. Similarly,
a button that toggles `(0, 2, 3)` becomes `0b1101`.

Here's the full solution:

```python
def solve(goal, state, tail, presses=0):
    global best

    if goal == state:
        best = min(best, presses)
        return

    if not tail:
        return

    h = tail[0]
    solve(goal, state ^ h, tail[1:], presses + 1)
    solve(goal, state, tail[1:], presses)
```

`goal` is the end state we want to reach, `state` is the current state,
initially 0. `tail` are the buttons we haven't tried pressing yet (initially
this contains all the buttons), and `presses` is the number of buttons we
pressed so far.

Part 2 was a lot more difficult. This is a linear optimization problem. Our
target is expressed by the joltages ${j_0, j_1, ... j_m}$. We have $n$ buttons.
A button press increments some joltages. Let's define a matrix $options$ such
that $options_(button, joltage)$ is $0$ if pressing the button $button$ doesn't
increase the joltage $joltage$ and $1$ if it does.

Then our problem becomes:

Given the system of equations

$$\begin{bmatrix}\mathrm{options}_{0,0}&\mathrm{options}_{1,0}&\cdots&\mathrm{options}_{n-1,0}\\\mathrm{options}_{0,1}&\mathrm{options}_{1,1}&\cdots&\mathrm{options}_{n-1,1}\\\vdots&\vdots&\ddots&\vdots\\\mathrm{options}_{0,m-1}&\mathrm{options}_{1,m-1}&\cdots&\mathrm{options}_{n-1,m-1}\end{bmatrix}\begin{bmatrix}x_0\\x_1\\\vdots\\x_{n-1}\end{bmatrix}=\begin{bmatrix}j_0\\j_1\\\vdots\\j_{m-1}\end{bmatrix}$$

find the smallest sum $x_0 + x_1 + ... x_{n-1}$ where all of the $x$s are
positive integers.

This is where I broke my rule of not taking dependencies on external libraries
and reached out for scipy. I used `milp` (Mixed-Integer Linear Programming):

```python
def solve(options, joltage):
    n = len(options)
    matrix = [[options[j][i] for j in range(n)]
         for i in range(len(joltage))]

    res = milp(
        c=[1] * n,
        constraints=LinearConstraint(matrix, joltage, joltage),
        bounds=Bounds(lb=[0] * n, ub=[max(joltage)] * n),
        integrality=[1] * n,
    )

    return sum([int(round(x)) for x in res.x])
```

`options` here is our matrix of which button increments which joltages and
`joltage` is the target state.

We invoke `milp` with coefficients 1 (`c`), specifying the linear constraint
`joltage <= matrix.dot(x) <= joltage` - we want the solution to be exactly
`joltage`. The `bounds` are between 0 and maximum joltage - we can press a
button at least 0 times and as most the target joltage times. The `integrality`
parameter enforces all $x$s are integers.

This produces the optimal vector.

## Day 11: Reactor

Problem statement is [here](https://adventofcode.com/2025/day/11).

The first part is easy. I implemented a `traverse` that starts from `you`
and recursively traverses all connected nodes. Whenever we hit the `out`
node we increment a running total:

```python
total = 0

def traverse(node):
    global total

    if node == "out":
        total += 1
        return

    for n in graph[node]:
        traverse(n)

traverse("you")
```

The graph we need to traverse is much larger in part 2. Here, I used a 2-step
approach. First, I sorted the nodes topologically:

```python
topological_order = []
visited = set()

def dfs(node):
    if node in visited:
        return
    visited.add(node)
    for neighbor in graph.get(node, []):
        dfs(neighbor)
    topological_order.append(node)

for node in graph:
    dfs(node)
```

From these, I generated the reverse of the graph:

```python
topological_order.reverse()

reverse_graph = {}
for node, neighbors in graph.items():
    for neighbor in neighbors:
        reverse_graph.setdefault(neighbor, []).append(node)
```

With these, we can compute the number of paths between a `start` and `end`
node with:

```python
def paths(start, end):
    paths = {node: 0 for node in graph}
    paths[start] = 1
    for node in topological_order[topological_order.index(start)+1:topological_order.index(end)+1]:
        paths[node] = sum(paths[prev] for prev in reverse_graph.get(node, []))
    return paths[end]
```

The solution is:

```python
paths("svr", "fft") * paths("fft", "dac") * paths("dac", "out")
```

## Day 12: Christmas Tree Farm

Problem statement is [here](https://adventofcode.com/2025/day/12).

This last one was more of a prank in my opinion. Reading the problem statement,
it's obvious the problem explodes combinatorially: we can rotate and flip
shapes, try to fit them together in various ways, the number of each type of
shape is different for each region. We obviously can't use backtracking for
this.

I first started looking at the inputs to see if there is maybe some optimal
combination of shapes that simplifies things, but the fact that we get different
numbers of each shape would make this not work.

Is there a case in which the shapes are guaranteed to fit without us having to
do any flipping/combining? Yes, there is - each shape is in a 3x3 grid, even if
it doesn't fully cover it. For a given region, if we can fit the pieces one
after another, making each take a 3x3 spot, then we can safely ignore any
rotations or packing. In other words, if the region is large enough, we have a
trivial solution.

Well, it turns out a large portion of the input is trivially solved like this.
As for the rest, let's look at the reverse - is there a case in which the shapes
are guaranteed *not* to fit regardless of how we combine them? Yes, there is -
for this case, we count the exact number of points a shape takes. So

```text
..#
.##
##.
```

takes 5 points, the 5 `#`. If the total number of points the shapes we want to
fit is larger than the number of points in the region, it means that regardless
of how we try to rotate and arrange our shapes, they will not fit. Well, it
turns out the rest of the input cases fall in this bucket.

So even though the general case explodes combinatorially and to my knowledge
there is no practical solution for it, the input cases given are all trivially
solvable as they are either way too small or large enough for us to compute the
fit without doing any work.

All in all, it was a fun set of problems. I am a bit bummed that this year
Advent of Code ran shorter than previous years. Problem 10 was strange, it was
the first one where I reached for scipy. Most problems I can solve without
external libraries but there was no way I was going to roll my own `milp`. And
problem 12 was straight up annoying - I spent an embarrassing amount of time to
arrive at the trivial solution. Eleven more months till the next Advent of Code!
