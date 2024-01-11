# Notes on Advent of Code 2023

I always have fun with [Advent of Code](https://adventofcode.com) every
December, and last year I did write [a blog post](https://vladris.com/blog/2023/01/07/notes-on-advent-of-code-2022.html)
covering some of the more interesting problems I worked through. I'll continue
the tradition this year.

I'll repeat my disclaimer from last time:

> **Disclaimer on my solutions**
>
> I use Python because I find it easiest for this type of coding. I treat
> solving these as a write-only exercise. I do it for the problem-solving bit,
> so I don't comment the code & once I find the solution I consider it "done" -
> I don't revisit and try to optimize even though sometimes I strongly feel
> like there is a better solution. I don't even share code between part 1 and
> part 2 - once part 1 is solved, I copy/paste the solution and change it to
> solve part 2, so each can be run independently. I also rarely use libraries,
> and when I do it's some standard ones like `re`, `itertools`, or `math`. The
> code has no comments and is littered with magic numbers and strange variable
> names. This is not how I usually code, rather my decadent holiday indulgence.
> I wasn't thinking I will end up writing a blog post discussing my solutions so
> I would like to apologize for the code being hard to read.

All my solutions are on my GitHub **[here](https://github.com/vladris/aoc)**.

This time around, I did use GitHub Copilot, with mixed results. In general, it
mostly helped with tedious work, like implementing the same thing to work in
different directions - there are problems that require we do something while
heading north, then same thing while heading east etc. I did also observe it
produce buggy code that I had to manually edit.

I'll skip over the first few days as they tend to be very easy.

## Day 9

Problem statement is [here](https://adventofcode.com/2023/day/9).

This is an easy problem, I just want to call out a shortcut: for part 2, to
exact same algorithm as in part 1 works if you first reverse the input. This was
a neat discovery that saved me a bunch of work.

## Day 10

Problem statement is [here](https://adventofcode.com/2023/day/10).

Part 1 was again very straightforward. I found part 2 a bit more interesting,
especially the fact that we can determine whether a tile is "inside" or
"outside" our loop by only looking at a single row (or column). We always start
"outside", then scan each tile. If we hit a `|`, then we toggle from "outside"
to "inside" and vice-versa. If we hit an `L` or a `F`, we continue while we're
on a `-` (these are all parts of our loop), and we stop on the `7` or `J`. If we
started on `L` and ended on `J` or started on `F` and eded on `7` - meaning the
pipe bends and turns back the way we came, we don't change our state. On the
other hand, if the pipe goes "down" from `L` to `7` or "up" from `F` to `J`,
then we toggle "outside"/"inside". For each non-pipe tile, if we're "inside", we
count it. Maybe this is obvious but it took me a bit to figure it out.

```python
def scan_line(ln):
    total, i, inside, start = 0, -1, False, None
    while i < len(grid[0]) - 1:
        i += 1
        if (ln, i) not in visited:
            if inside:
                total += 1
        else:
            if grid[ln][i] == '|':
                inside = not inside
                continue
            
            # grid[ln][i] in 'LF'
            start = grid[ln][i]
            i += 1
            while grid[ln][i] == '-':
                i += 1

            if start == 'L' and grid[ln][i] == '7' or \
               start == 'F' and grid[ln][i] == 'J':
               inside = not inside
    return total
```

In the code above, `visited` tracks pipe segments (as opposed to tiles that are
not part of the pipe).

## Day 11

Problem statement is [here](https://adventofcode.com/2023/day/11).

Day 11 was easy, so not much to discuss. Use Manhattan distance for part 1 and
in part 2, just add `999999` for every row or column crossed that doesn't
contain any galaxies.

## Day 12

Problem statement is [here](https://adventofcode.com/2023/day/12).

Part 1 was very easy.

Part 2 was a bit harder because just trying out every combination takes forever
to run. I initially tried to do something more clever around deciding when to
turn a `?` into `#` or `.` depending on what's around it, where we are in the
sequence, etc. But ultimately it turns out just adding memoization made the
combinatorial approach run very fast.

## Day 13

Problem statement is [here](https://adventofcode.com/2023/day/13).

This was a very easy one, so I won't cover it.

## Day 14

Problem statement is [here](https://adventofcode.com/2023/day/14).

This was easy but part 2 was tedious, having to implement `tilt` functions for
various directions. This is where Copilot saved me a bunch of typing.

Once we have the `tilt` functions, we can implement a `cycle` function that
tilts things north, then west, then south, then east. Finally, we need a bit of
math to figure out the final position: we save the state of the grid after each
cycle and as soon as we find a configuration we encountered before, it means we
found our cycle. Based on this, we know how many steps we have before the cycle,
what the length of the cycle is, so we can compute the state after 1000000000
cycles:

```python
pos = []
while (state := cycle()) not in pos:
    pos.append(state)

lead, loop = pos.index(state), len(pos) - pos.index(state)
d = (1000000000 - lead) % loop
```

With this, we need to count the load of the north support beams for the grid we
have at `pos[lead + d - 1]`.

## Day 15

Problem statement is [here](https://adventofcode.com/2023/day/15).

Another very easy one that I won't cover.

## Day 16

Problem statement is [here](https://adventofcode.com/2023/day/16).

This one was also easy and tedious, as we have to handle the different types of
reflections. Another one where Copilot saved me a lot of typing.

## Day 17

Problem statement is [here](https://adventofcode.com/2023/day/17).

### Part 1

This was a fairly straightforward depth-first search, where we keep a cache of
how much heat loss we have up to a certain point. The one interesting
complication is that we can only move forward 3 times. In the original
implementation, I keyed the cache on grid coordinates + direction we're going
in + how many steps we already took in that direction. This worked in
reasonable time.

### Part 2

In part 2, we now have to move at least 4 steps in one direction and at
most 10. The cache I used in part 1 doesn't work that well anymore. On the
other hand, I realized that rather than keeping track of direction and how
many steps we took in that direction so far, I can model this differently: we
are moving either horizontally or vertically. If we're at some point and moving
horizontally, we can expand our search to all destination points (from 4 to 10
away horizontally or vertically) and flip the direction. For example, if we
just moved horizontally to the right, we won't move further to the right as we
already covered all those cases, and we won't move back left as the crucible
can't turn 180 degrees. That means the only possible directions we can take are
up or down in this case, meaning since we just moved horizontally, we now have
to move vertically.

This makes our cache much smaller: our key is the coordinates of the cell and
the direction we were moving in. This also makes the depth-first search
complete very fast.

```python
best, end = {}, 1000000

def search(x, y, d, p):
    global end

    if p >= end:
        return

    if x == len(grid) - 1 and y == len(grid[0]) - 1:
        if p < end:
            end = p
        return

    if (x, y, d) in best and best[(x, y, d)] <= p:
        return

    best[(x, y, d)] = p
    
    if d != 'H':
        if x + 3 < len(grid[x]):
            pxr = p + grid[x + 1][y] + grid[x + 2][y] + grid[x + 3][y]
            for i in range(4, 11):
                if x + i < len(grid):
                    pxr += grid[x + i][y]
                    search(x + i, y, 'H', pxr)

        if x - 3 >= 0:
            pxl = p + grid[x - 1][y] + grid[x - 2][y] + grid[x - 3][y]
            for i in range(4, 11):
                if x - i >= 0:
                    pxl += grid[x - i][y]
                    search(x - i, y, 'H', pxl)

    if d != 'V':
        if y + 3 < len(grid[0]):
            pyd = p + grid[x][y + 1] + grid[x][y + 2] + grid[x][y + 3]
            for i in range(4, 11):
                if y + i < len(grid[0]):
                    pyd += grid[x][y + i]
                    search(x, y + i, 'V', pyd)

        if y - 3 >- 0:
            pyu = p + grid[x][y - 1] + grid[x][y - 2] + grid[x][y - 3]
            for i in range(4, 11):
                if y - i >= 0:
                    pyu += grid[x][y - i]
                    search(x, y - i, 'V', pyu)
```

I realized this approach actually applies well to part 1 too, and retrofitted it
there. The only difference is instead of expanding to the cells +4 to +10 in a
direction, we expand to the cells +1 to +3.

## Day 18

Problem statement is [here](https://adventofcode.com/2023/day/18).

### Part 1

The first part is easy - we plot the input on a grid, then flood fill to find
the area.

In the below code, `dig` is the input, processed as a tuple of direction and
number of steps:

```python
x, y, grid = 0, 0, {(0, 0)}
for dig in digs:
    match dig[0]:
        case 'U':
            for i in range(dig[1]):
                y -= 1
                grid.add((x, y))
        case 'R':
            for i in range(dig[1]):
                x += 1
                grid.add((x, y))
        case 'D':
            for i in range(dig[1]):
                y += 1
                grid.add((x, y))
        case 'L':
            for i in range(dig[1]):
                x -= 1
                grid.add((x, y))

x, y = min([x for x, _ in grid]), min([y for _, y in grid])
while (x, y) not in grid:
    y += 1

queue = [(x + 1, y + 1)]
while queue:
    x, y = queue.pop(0)

    if (x, y) in grid:
        continue

    grid.add((x, y))
    queue += [(x + 1, y), (x - 1, y), (x, y + 1), (x, y - 1)]

print(len(grid))
```

### Part 2

Part 2 is trickier, as the number are way larger and the same flood fill
algorithm won't work. My approach was to divide the area into rectangles: as we
process all movements, we end up with a set of `(x, y)` tuples of points where
our line changes direction. If we sort all the `x` coordinates and all `y`
coordinates independently, we end up with a grid where we can treat each pair of
subsequent `x`s and `y`s as describing a rectangle on our grid.

```python
x, y, points = 0, 0, [(0, 0)]
for dig in digs:
    match dig[0]:
        case 0: x += dig[1]
        case 1: y += dig[1]
        case 2: x -= dig[1]
        case 3: y -= dig[1]

    if dig[1] < 10:
        print(dig[1])
    points.append((x, y))

xs, ys = sorted({x for x, _ in points}), sorted({y for _, y in points})
```

Where `digs` above represents the input, processed as before into direction and
number of steps tuples.

Now `points` contains all the connected points we get following the directions,
which means a pair of subsequent points describes a line. Once we have this, we
can start a flood fill in one of the rectangles and proceed as follows: if there
is a north boundary, meaning we have a line between our top left and top right
coordinates, then we don't recurse north; otherwise we go to the rectangle north
of our current rectangle and repeat the algorithm there. Same for east, south,
west.

Since we have to consider each point in the terrain in our area calculation, we
need to be careful how we measure the boundaries of each rectangle so we don't
double-count or omit points. To ensure this, my approach was that for each
rectangle we count, we count an extra line north (if there is no boundary) and
an extra line east (if there is no boundary). If there's neither a north nor an
east boundary, then we add 1 for the north-east corner. This should ensure we
don't double-count, as each rectangle only considers its north and east
boundaries, and we don't miss anything, as any rectangle without a boundary will
count the additional points. What remains is the perimeter of our surface, which
we add it at the end. The explanations might sound convoluted, but the code is
very easy to understand:

```python
queue, total, visited = [(1, 1)], 0, set()
while queue:
    x, y = queue.pop(0)

    e = min([i for i in xs if i > x])
    s = max([i for i in ys if i < y])
    w = max([i for i in xs if i < x])
    n = min([i for i in ys if i > y])

    if (n, e) in visited:
        continue
    visited.add((n, e))

    total += (e - w - 1) * (n - s - 1)

    found_n, found_s, found_e, found_w = False, False, False, False
    for l1, l2 in zip(points, points[1:]):
        if l1[1] == l2[1]:
            if l1[1] == n and (l1[0] < x < l2[0] or l2[0] < x < l1[0]):
                found_n = True
            if l1[1] == s and (l1[0] < x < l2[0] or l2[0] < x < l1[0]):
                found_s = True
        elif l1[0] == l2[0]:
            if l1[0] == e and (l1[1] < y < l2[1] or l2[1] < y < l1[1]):
                found_e = True
            if l1[0] == w and (l1[1] < y < l2[1] or l2[1] < y < l1[1]):
                found_w = True
                
    if not found_n:
        total += e - w - 1
        queue.append((x, n + 1))
    if not found_s:
        queue.append((x, s - 1))
    if not found_e:
        total += n - s - 1
        queue.append((e + 1, y))
    if not found_w:
        queue.append((w - 1, y))

    if not found_n and not found_e:
        if (e, n) not in points:
            total += 1

total += sum([dig[1] for dig in digs])
```

## Day 19

Problem statement is [here](https://adventofcode.com/2023/day/19).

### Part 1

For the first part, we can process rule by rule.

### Part 2

For the second part, start with bounds: `(1, 4000)` for all of `xmas`. Then at
each decision point, recurse updating bounds. Whenever we hit an `A`, add the
bounds to the list of accepted bounds.

Bounds are guaranteed to never overlap, by definition.

```python
accepts = []

def execute_workflow(workflow_key, bounds):
    workflow = workflows[workflow_key]
    for rule in workflow:
        if rule == 'A':
            accepts.append(bounds)
            return
        if rule == 'R':
            return
        if rule in workflows:
            execute_workflow(rule, bounds)
            return

        check, next_workflow = rule.split(':')
        if '<' in check:
            key, val = check.split('<')
            nb = bounds.copy()
            nb[key] = (nb[key][0], int(val) - 1)
            bounds[key] = (int(val), bounds[key][1])
        elif '>' in check:
            key, val = check.split('>')
            nb = bounds.copy()
            nb[key] = (int(val) + 1, nb[key][1])
            bounds[key] = (bounds[key][0], int(val))

        execute_workflow(next_workflow, nb)

execute_workflow('in', {'x': (1, 4000), 'm': (1, 4000), 'a': (1, 4000), 's': (1, 4000)})
```

This gives us all accepted ranges for each of `x`, `m`, `a`, and `s`.

## Day 20

Problem statement is [here](https://adventofcode.com/2023/day/20).

### Part 1

For the first part, we can model the various module types as classes with a
common interface and different implementations. Since one of the requirements is
to process pulses in the order they are sent, we will use a queue rather than
have objects call each other based on connections. So rather than module `A`
directly calling connected module `B` when it receives a signal (which would
cause out-of-order processing), model `A` will just queue a signal for module
`B`, which will be processed once the signals queued before this one are already
processed.

I won't share the code here as it is straightforward. You can find it on my
GitHub.

### Part 2

This one was one of the most interesting problems this year. Simply simulating
button presses wouldn't work. I ended up dumping the diagram as a dependency
graph and it looks like the only module that signals `rx` is a conjunction
module with multiple inputs.

Conjunction modules emit a low pulse when they remember high pulses being sent
by all their connected inputs. In this case, we can simulate button presses and
keep track when each input to this conjunction module emits a high pulse. Then
we compute the least common multiple of these to determine when the `rx` module
will get a low signal.

My full solution is
[here](https://github.com/vladris/aoc/blob/master/2023/20/02.py), though I'm
still pretty sure it is topology-dependent. Meaning we might have a different
set up where the inputs to this conjunction model are not fully independent,
which might make LCM not return the correct answer.

## Day 21

Problem statement is [here](https://adventofcode.com/2023/day/21).

### Part 1

Part 1 is trivial, we can easily simulate 64 steps and count reachable spots.

### Part 2

The second part is much more tricky - this is actually the problem I spent the
most time on. Since the garden is infinite, and we are looking for a very high
number of steps, we can't use the same approach as in part 1 to simply simulate
moves.

Let's now call a "tile" a repetition of the garden on our infinite grid. Say we
start with the garden at `(0, 0)`. Then as we expand beyond its bounds, we reach
tiles `(-1, 0)`, `(1, 0)`, `(0, -1)`, `(0, 1)`, which are repetitions of our
initial garden.

The two observations that helped here were:

1. Once we reach all possible spots in a garden, following steps just cycle
   between the same two sets of reachable spots. Meaning once we spend enough
   time in a garden, we know how many steps are reachable in that particular
   garden by just looking at the modulo of total number of steps.
2. As the number of steps increases over the infinitely repeating garden, there
   is a pattern to how the covered area grows. This is a diamond shape where the
   center is always fully covered garden tiles (see the first observation above)
   and the surrounding tiles are at various stages of being visited.

In fact, after we grow beyond the first 4 surrounding tiles, it seems like the
garden grows with a periodicity of the size of the garden. Meaning every
`len(grid)` steps, we reach new tiles. There are a few cases to consider -
north, east, south, west, diagonals.

My approach was to do a probe - simulate the first few steps and record the
results.

```python
def probe():
    dx, dy = len(grid) // 2, len(grid[0]) // 2
    tiles, progress = {(dx, dy)}, {(0, 0): {0: 1}}
    
    i = 0
    while len(progress) < 41:
        i += 1
        new_tiles = set()
        for x, y in tiles:
            if grid[(x - 1) % len(grid)][y % len(grid[0])] != '#':
                new_tiles.add((x - 1, y))
            if grid[(x + 1) % len(grid)][y % len(grid[0])] != '#':
                new_tiles.add((x + 1, y))
            if grid[x % len(grid)][(y - 1) % len(grid[0])] != '#':
                new_tiles.add((x, y - 1))
            if grid[x % len(grid)][(y + 1) % len(grid[0])] != '#':
                new_tiles.add((x, y + 1))

        tiles = new_tiles

        for x, y in tiles:
            sq_x, sq_y = x // len(grid), y // len(grid[0])
            if (sq_x, sq_y) not in progress:
                progress[(sq_x, sq_y)] = {}
            if i not in progress[(sq_x, sq_y)]:
                progress[(sq_x, sq_y)][i] = 0
            progress[(sq_x, sq_y)][i] += 1

    return progress
```

Here `progress` keeps track, for each tile (keyed as set of `(x, y)` coordinates
offset from `(0, 0)`), of how many spots are reachable at a given time. I run
this until `progress` grows enough for the repeating pattern to show - because
we start from the center of a garden but in all other tiles we enter from a
side, it takes a couple of iterations for the pattern to stabilize. My guess is
this probe could be smaller with some better math, but that's what I have.

With this, given a number of steps, we can reduce it using `steps % len(grid)`
to a smaller value we can loop in our `progress` record. The reasoning being, if
the pattern repeats, it doesn't really matter whether we are 3 steps into tile
`(-1000, 0)` or 3 steps into tile `(-3, 0)`.

The tedious part was determining the right offsets and special cases when
computing the total number of squares. For example, even for the tiles that are
fully covered, we'll have a subset where tiles are on the "odd" state of squares
and a subset where tiles are on the “even" state.

I ended up with the following formula (which might still be buggy, but seemed to
have worked for my input):

```python
def at(x, y, step):
    return progress[(x, y)][step] if step in progress[(x, y)] else 0


def count(steps):
    even, odd = (1, 0) if steps % 2 == 0 else (0, 1)

    for i in range(1, steps // len(grid)):
        if steps % 2 == 0:
            if i % 2 == 0:
                even += 4 * i
            else:
                odd += 4 * i
        else:
            if i % 2 == 0:
                odd += 4 * i
            else:
                even += 4 * i

    total = even * at(0, 0, len(grid) * 2) + odd * at(0, 0, len(grid) * 2 + 1)

    total += at(-3, 0, len(grid) * 3 + steps % len(grid))
    total += at(3, 0, len(grid) * 3 + steps % len(grid))
    total += at(0, -3, len(grid) * 3 + steps % len(grid))
    total += at(0, 3, len(grid) * 3 + steps % len(grid))

    i = steps // len(grid) - 1

    total += i * at(-1, -1, len(grid) * 2 + steps % len(grid))
    total += i * at(-1, 1, len(grid) * 2 + steps % len(grid))
    total += i * at(1, -1, len(grid) * 2 + steps % len(grid))
    total += i * at(1, 1, len(grid) * 2 + steps % len(grid))
    
    i += 1
    
    total += i * at(-2, -1, len(grid) * 2 + steps % len(grid))
    total += i * at(-2, 1, len(grid) * 2 + steps % len(grid))
    total += i * at(2, -1, len(grid) * 2 + steps % len(grid))
    total += i * at(2, 1, len(grid) * 2 + steps % len(grid))
    
    return total
```

I'm covering all inner "even" and “odd" tiles, then the directly north, east,
south, and west tiles, then two layers of diagonals. Again, I have a feeling
this could be simpler, but I didn't bother to optimize it further.

## Day 22

Problem statement is [here](https://adventofcode.com/2023/day/22).

### Part 1

For part one, we sort bricks by `z` coordinate (ascending), then we make each
brick "fall". We do this by decrementing their `z` coordinate and checking
whether they intersect with any other brick.

```python
def intersect(brick1, brick2):
    if brick1[0].x > brick2[1].x or brick1[1].x < brick2[0].x:
        return False
    
    if brick1[0].y > brick2[1].y or brick1[1].y < brick2[0].y:
        return False
    
    if brick1[0].z > brick2[1].z or brick1[1].z < brick2[0].z:
        return False
    
    return True


def slide_down(brick, delta):
    return (Point(brick[0].x, brick[0].y, brick[0].z - delta), Point(brick[1].x, brick[1].y, brick[1].z - delta))


def fall(brick):
    if min(brick[0].z, brick[1].z) == 1:
        return 0

    result, orig = 0, brick
    while True:
        brick = slide_down(brick, 1)
        for b in bricks:
            if b == orig:
                continue

            if intersect(brick, b):
                return result

        result += 1
        if min(brick[0].z, brick[1].z) == 1:
            return result


bricks = sorted(bricks, key=lambda b: min(b[0].z, b[1].z))

for i, brick in enumerate(bricks):
    if delta := fall(brick):
        bricks[i] = slide_down(brick, delta)
```

Once every brick that could fall has fallen to its final position, we need to
find the "critical" bricks - the bricks that are the only support for some other
bricks. We do this by shifting down each brick again 1 `z` and determining how
many bricks it intersects with. If a shifted brick only intersects with one
other brick, that is a “critical" brick, so we add it to our set of “critical"
support bricks. All other bricks can be safely removed.

```python
critical = set()
for brick in bricks:
    if brick[0].z == 1 or brick[1].z == 1:
        continue

    supported_by = []
    nb = slide_down(brick, 1)
    for i, b in enumerate(bricks):
        if brick == b:
            continue

        if intersect(nb, b):
            supported_by.append(i)

    if len(supported_by) == 1:
        critical.add(supported_by[0])

print(len(bricks) - len(critical))
```

### Part 2

In part 2, we need to figure out which bricks is each brick supported by. We can
use a similar algorithm to part 1, where we shift `z` by 1 and check which
bricks we intersect. Then we can build a dependency graph of which bricks is
supported by which other bricks.

```python
supported_by = {}
for i, brick in enumerate(bricks):
    supported_by[i] = set()

    if brick[0].z == 1 or brick[1].z == 1:
        continue

    nb = slide_down(brick, 1)
    for j, b in enumerate(bricks):
        if i == j:
            continue

        if intersect(nb, b):
            supported_by[i].add(j)
```

Then for each brick we remove, we can walk the "supported by" dependencies to
determine which bricks would fall and would, in turn, cause other bricks to
fall, without having to actually simulate falling.

```python
def count_falling(i):
    sup = {k: supported_by[k].copy() for k in supported_by.keys()}
    queue, removed = [i], set()
    while queue:
        i = queue.pop(0)
        
        if i in removed:
            continue
        removed.add(i)

        for j in sup:
            if i in sup[j]:
                sup[j].remove(i)
                if len(sup[j]) == 0:
                    queue.append(j)

    return len(removed) - 1


print(sum(count_falling(i) for i in range(len(supported_by))))
```

## Day 23

Problem statement is [here](https://adventofcode.com/2023/day/23).

The main insight here for both part 1 and part 2 is that we can model the paths
as a graph where each intersection (decision point) is a vertex and the paths
between intersections are edges. With this representation, we simply need to
find the longest path between our starting point and our end point.

In part 1, we have a directed graph, as right before hitting each intersection,
we have a `><^v` constraint, making the path one-way. In part 2, we have an
undirected graph.

Note that the longest path problem in a graph is harder than the shortest path
problem. That said, we are dealing with extremely small graphs.

## Day 24

Problem statement is [here](https://adventofcode.com/2023/day/24).

### Part 1

Part 1 was fairly straightforward: for each pair of lines, solve the equation to
find where they meet and check if within bounds (when lines are not parallel).

Since each line is described by a point $(x_{origin}, y_{origin})$ and a vector $(dx, dy)$, we
can represent them as

$$\begin{cases}
x = x_{origin} + dx * t \\
y = y_{origin} + dy * t
\end{cases}$$

Then the lines intersect when

$$\begin{cases}
x_1 + dx_1 * t_1 = x_2 + dx_2 * t_2 \\
y_1 + dy_1 * t_1 = y_2 + dy_2 * t_2
\end{cases}$$

We know all of $(x_1, y_1), (dx_1, dy_1), (x_2, y_2), (dx_2, dy_2)$ so we solve
for $t_1$ and $t_2$.

```python
def intersect(p1, v1, p2, v2):
    if v1.dx / v1.dy == v2.dx / v2.dy:
        return None, None

    t2 = (v1.dx * (p2.y - p1.y) + v1.dy * (p1.x - p2.x)) / (v2.dx * v1.dy - v2.dy * v1.dx)
    t1 = (p2.y + v2.dy * t2 - p1.y) / v1.dy

    return t1, t2
```

Once we have `t1` and `t2`, we need to check both are positive (so intersection didn't happen in the past), and make sure the intersection point, which is either `x1 + dx1 * t1`, `y1 + dx1 * t1` or `x2 + dx2 * t2`, `y2 + dx2 * t2`, is within our bounds (at least 200000000000000 and at most 400000000000000).

If that's the case, then we found an intersection and we can add it to the total.

### Part 2

Part 2 was really fun. We now have 3 dimensions, so a line is represented as

$$\begin{cases}
x = x_{origin} + dx * t \\
y = y_{origin} + dy * t \\
z = z_{origin} + dz * t
\end{cases}$$

We need to find a line (the trajectory of our rock) that intersects each line in
our input at a different time, such that for some $t$ and line $l$, we have

$$\begin{cases}
x_{origin_{l}} + dx_l * t = x_{origin_{rock}} + dx_{rock} * t \\
y_{origin_{l}} + dy_l * t = y_{origin_{rock}} + dy_{rock} * t \\
z_{origin_{l}} + dz_l * t = z_{origin_{rock}} + dz_{rock} * t
\end{cases}$$

One way to solve this is using linear algebra. If we take 3 different hailstorms
and our rock, we end up with the following set of equations:

$$\begin{cases}
x_{origin_{1}} + dx_1 * t_1 = x_{origin_{rock}} + dx_{rock} * t_1 \\
y_{origin_{1}} + dy_1 * t_1 = y_{origin_{rock}} + dy_{rock} * t_1 \\
z_{origin_{1}} + dz_1 * t_1 = z_{origin_{rock}} + dz_{rock} * t_1 \\
x_{origin_{2}} + dx_2 * t_2 = x_{origin_{rock}} + dx_{rock} * t_2 \\
y_{origin_{2}} + dy_2 * t_2 = y_{origin_{rock}} + dy_{rock} * t_2 \\
z_{origin_{2}} + dz_2 * t_2 = z_{origin_{rock}} + dz_{rock} * t_2 \\
x_{origin_{3}} + dx_3 * t_3 = x_{origin_{rock}} + dx_{rock} * t_3 \\
y_{origin_{3}} + dy_3 * t_3 = y_{origin_{rock}} + dy_{rock} * t_3 \\
z_{origin_{3}} + dz_3 * t_3 = z_{origin_{rock}} + dz_{rock} * t_3
\end{cases}$$

In the above system, we know all of the starting points and vectors of the
hailstorms. Our unknowns are $t_1, t_2, t_3, x_{origin_{rock}},
y_{origin_{rock}}, z_{origin_{rock}}, dx_{rock}, dy_{rock}, dz_{rock}$. That's
9 unknowns to 9 equations, so it should be solvable.

While this approach works, I didn't want to use a numerical library to solve
this (I'm trying to keep dependencies at a minimum), and implementing the math
from scratch was a bit too much for me. I thought of a different approach: as
long as we can find a rock trajectory that intersects the first couple of
hailstorms at the right times, we most likely found our solution.

$$\begin{cases}
x_{origin_{rock}} + dx_{rock} * t_1 = x_1 + dx_1 * t_1 \\
y_{origin_{rock}} + dy_{rock} * t_1 = y_1 + dy_1 * t_1 \\
x_{origin_{rock}} + dx_{rock} * t_2 = x_2 + dx_2 * t_2 \\
y_{origin_{rock}} + dy_{rock} * t_2 = y_2 + dy_2 * t_2
\end{cases}$$

If we solve this for $t_1$ and $t_2$, we can then easily determine
$z_{origin_{rock}}$ and $dz_{rock}$.

In the above set of equations, we have too many unknowns: $x_{origin_{rock}},
dx_{rock}, y_{origin_{rock}}, dy_{rock}, t_1, t_2$. We can reduce this number
by trying out different values for a couple of these unknowns. While the ranges
of possible values for $x_{origin_{rock}}, y_{origin_{rock}}, t_1, t_2$ are
very large, so unfeasible to cover, $dx_{origin}$ and $dy_{origin}$ ranges
should be small - if these values are large, our rock will quickly shoot past
all the other hailstorms.

My approach was to try all possible values between -1000 and 1000 for both of
these, then see if we can find $x_{origin_{rock}}, y_{origin_{rock}}, t_1,
t_2$ such that these intersect the first two hailstorms. If we do, we then
find $z_{origin_{rock}}, dz_{rock}$ (easy to find since now we know $t_1, t_2$).
We have an additional helpful constraint: the origin coordinates of the rock
need to be integers.

Then we just need to check that indeed for the given $(x_{origin_{rock}},
y_{origin_{rock}}, z_{origin_{rock}})$ and $(dx_{rock}, dy_{rock}, dz_{rock})$,
for each hailstorm, there is a time $t_i$ when they intersect.

Here is the code:

```python
def find(rng):
    for dx in range(-rng, rng):
        for dy in range(-rng, rng):
            x1, y1, z1 = hails[0][0]
            dx1, dy1, dz1 = hails[0][1]
            x2, y2, z2 = hails[1][0]
            dx2, dy2, dz2 = hails[1][1]

            # x + dx * t1 = x1 + dx1 * t1
            # y + dy * t1 = y1 + dy1 * t1
            # x + dx * t2 = x2 + dx2 * t2
            # y + dy * t2 = y2 + dy2 * t2

            # x = x1 + t1 * (dx1 - dx)        
            # t1 = (x2 - x1 + t2 * (dx2 - dx)) / (dx1 - dx)
            # y = y1 + (x2 - x1 + t2 * (dx2 - dx)) * (dy1 - dy) / (dx1 - dx)
            # t2 = ((y2 - y1) * (dx1 - dx) - (dy1 - dy) * (x2 - x1)) / ((dy1 - dy) * (dx2 - dx) + (dy - dy2) * (dx1 - dx))

            if (dy1 - dy) * (dx2 - dx) + (dy - dy2) * (dx1 - dx) == 0:
                continue

            t2 = ((y2 - y1) * (dx1 - dx) - (dy1 - dy) * (x2 - x1)) / ((dy1 - dy) * (dx2 - dx) + (dy - dy2) * (dx1 - dx))

            if not t2.is_integer() or t2 < 0:
                continue

            if (dx1 - dx) == 0:
                continue

            y = y1 + (x2 - x1 + t2 * (dx2 - dx)) * (dy1 - dy) / (dx1 - dx)

            if not y.is_integer():
                continue

            t1 = (x2 - x1 + t2 * (dx2 - dx)) / (dx1 - dx)

            if not t1.is_integer() or t1 < 0:
                continue

            x = x1 + t1 * (dx1 - dx)        

            # z + dz * t1 = z1 + dz1 * t1
            # z + dz * t2 = z2 + dz2 * t2        

            # dz = (z1 + dz1 * t1 - z2 - dz2 * t2) / (t1 - t2)
            # z = z1 + dz1 * t1 - dz * t1

            if t1 == t2:
                continue

            dz = (z1 + dz1 * t1 - z2 - dz2 * t2) / (t1 - t2)

            if not dz.is_integer():
                continue

            z = z1 + dz1 * t1 - dz * t1
```

In the above `x`, `y`, `z`, `dx`, `dy`, `dz` are the rock's origin and vector.

The final step (omitted from the code sample for brevity), is to confirm that
for the given origin and vector, we end up eventually intersecting all other
hailstorms.

I really enjoyed this problem as it made me work through the math.

## Day 25

Problem statement is [here](https://adventofcode.com/2023/day/25).

I liked this problem. It turned out to be a variation of the [minimum cut
problem](https://en.wikipedia.org/wiki/Minimum_cut). Trying out all possible
permutations of nodes would take way too much time. The algorithm I used keeps
track of a set of *visited* nodes - one of the two components. Then at each
step, we add a new node to this set by selecting the most connected node to this
component (meaning the node that has most edges incoming from visited nodes).

`most_connected()` determines which node we want to pick next:

```python
def most_connected(visited):
    best_n, best_d = None, 0
    for n in graph:
        if n in visited:
            continue

        neighbors = sum(1 for v in graph[n] if v in visited)
        if neighbors > best_d:
            best_n, best_d = n, neighbors

    return best_n
```

Then we keep going until our component has exactly 3 outgoing edges to nodes
that haven't ben visited yet:

```python
def find_components():
    start = list(graph.keys())[0]
    visited = {start}
    while len(visited) < len(graph):
        total = 0
        for n in visited:
            total += sum(1 for v in graph[n] if v not in visited)
        
        if total == 3:
            return visited

        n = most_connected(visited)
        visited.add(n)
```

That's where we need to make the cut. We just need to multiply `len(visited)`
with `len(graph) - len(visited)` to find our answer.

I personally found the most difficult problems to be part 2 of day 20, 21, 24
and the one and only part of day 25. All of these took me a bit to figure out.
That said, Advent of Code is always a nice holiday past-time and I can't wait
for the 2024 iteration.
