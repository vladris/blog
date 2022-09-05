# Timsort

I wrote before about the inherent complexity of the real world and how
software that behaves well in the real world must necessarily take on
some complexity ([Time and
Complexity](https://vladris.com/blog/2020/01/19/time-and-complexity.html)).
A lot of the software engineering best practices try to reduce or
eliminate the *accidental complexity* of large systems (making things
more complicated than they should be). But we don't live in a perfect
world, so modeling it using software requires some *inherent complexity*
in the software, to reflect reality. One of the algorithms which
perfectly illustrates this is the Timsort sorting algorithm.

Timsort is an algorithm developed by Tim Peters in 2002 to replace
Python's previous sorting algorithm. It has since been adopted
inJava's OpenJDK, the V8 JavaScript engine, and the Swift and Rust
languages. This is a testament that Timsort is a performant sort.

Timsort is a *stable sorting algorithm*, which means it will never
changes the relative order of equal elements. This is an important
property in certain situations. This is not important when sorting
numbers, but becomes important when sorting objects with custom
comparisons.

But in 2002 we already had plenty of well known sorting algorithms which
were quite efficient. How did Timsort manage to outperform these?

## Merging Runs

The key insight of Timsort is that in the real world, many lists of
elements that require sorting contain subsequences of elements that are
already sorted. These are called *runs* and tend to appear naturally.
For example, in the list `[5, 2, 3, 4, 9, 1, 6, 8, 10, 7]` we have two
runs: `[2, 3, 4]` and `[1, 6, 8, 10]`.

If we know runs will show up more often than not in our input, how can
we best leverage this to our advantage, and avoid extraneous comparisons
and data movement?

Timsort starts by finding the minimum "accepted" run length for a
given input. This doesn't have anything to do with the content of the
input, rather it is a function of the size of the input. More on this
later.

Then we do a single pass over the array and identify consecutive runs.
If the next *minimum accepted run length* elements are not already
sorted (they don't form a run), we sort them using insertion sort (so
they do end up as a run). We push these runs on a stack, then we then
merge pairs of them until we end up with a single run, which is our
sorted list.

## A Simple Implementation

Let's start with a simple sketch implementation. We'll use Python
since it is expressive and it makes it easier to focus on the algorithm
rather than syntax around it.

``` python
MIN_MERGE = 4

def sort(arr): 
    lo, hi = 0, len(arr) 
    stack = []
    nRemaining = hi
    minRun = MIN_MERGE

    while nRemaining > 0:
        runLen = min(nRemaining, minRun)
        insertionSort(arr, lo, lo + runLen)
        stack.append((lo, runLen))

        lo += runLen
        nRemaining -= runLen

    while len(stack) > 1:
        base2, len2 = stack.pop()
        base1, len1 = stack.pop()
        merge(arr, base1, base2, base2 + len2)
        stack.append((base1, len1 + len2))
```

First, we initialize a few variables:

``` python
lo, hi = 0, len(arr) 
stack = []
nRemaining = hi
minRun = MIN_MERGE
```

`MIN_MERGE` represents the minimum number of elements we want to merge,
and is a constant. We'll talk more about this once we look at some
optimizations later on.

`lo` and `hi` represent the range in the array we will operate on. Note
ranges are always half-open (`arr[lo]` included, `arr[hi]` excluded,
potentially out of bounds). `stack` is the run stack, `nRemaining` is
the number of elements we still need to process. `minRun` is the minimum
run length. For this first iteration, we'll just use `MIN_MERGE`.

Next, we traverse the array and come up with our runs:

``` python
while nRemaining > 0:
        runLen = min(nRemaining, minRun)
        insertionSort(arr, lo, lo + runLen)
        stack.append((lo, runLen))

        lo += runLen
        nRemaining -= runLen
```

Our run in this case will be the minimum between `minRun` and the
remaining elements of the array (so for the final run, we don't go out
of bounds). We sort the run using `insertionSort`, then we push the run
start index and length onto the stack. We advance `lo` by the length of
the run and we similarly decrement `nRemaining`, the number of elements
still to be processed.

Next, we merge the runs:

``` python
while len(stack) > 1:
    base2, len2 = stack.pop()
    base1, len1 = stack.pop()
    merge(arr, base1, base2, base2 + len2)
    stack.append((base1, len1 + len2))
```

We pop 2 runs from the top of the stack, merge them, and push the new
run back onto the stack. With this basic implementation, a stack is
technically not really needed, but I'm trying to preserve the general
shape of the optimized solution.

We called a couple of helper functions: `insertionSort` and `merge`.
Here is `insertionSort`:

``` python
def insertionSort(arr, lo, hi): 
    i = lo + 1

    while i < hi:
        elem = arr[i]
        j = i - 1

        while elem < arr[j] and j >= lo:
        j -= 1

        arr.pop(i)
        arr.insert(j + 1, elem)

        i += 1
```

[Insertion sort](https://en.wikipedia.org/wiki/Insertion_sort) traverses
the array from the lower bound + 1 to the higher bound and maintains the
invariant that all elements preceding `i` are sorted. So for any element
`arr[i]`, we find a spot `j` in the range `[lo, i)` where this element
should fit. We then insert it there and shift the remaining elements in
`[j + 1, i)` one spot to the right. Note this algorithm is quite
inefficient on large data sets, but performs well on small inputs.

Our merge algorithm is:

``` python
def merge(arr, lo, mid, hi):
    t = arr[lo:mid]
    i, j, k = lo, mid, 0

    while k < mid - lo and j < hi:
        if t[k] < arr[j]:
            arr[i] = t[k]
            k += 1
        else:
            arr[i] = arr[j]
            j += 1
        i += 1

    if k < mid - lo:
        arr[i:hi] = t[k:mid - lo]
```

We are merging the consecutive (sorted) ranges `[lo, mid)` and
`[mid, hi)`. One way to do this (which our implementation uses), is to
copy `[lo, mid)` to a temporary buffer `t`. We then traverse the
`[mid, hi)` range with `j` and the buffer with `k`. We pick the smallest
of `t[k]` and `arr[j]` to insert at `arr[i]` (incrementing the
corresponding index), then we increment `i`. At some point, either `j`
or `k` reaches the end. If `j` makes it to the end first, it means we
still have some elements in `t` we need to copy over. If `k` makes it to
the end first, we don't need to do anything: the remaining elements in
`[j, hi)` are where they are supposed to be.

We now have a full implementation of a very simple Timsort. If we run it
on the `[5, 2, 3, 4, 9, 1, 6, 8, 10, 7]` input, the following steps take
place:

* We pick up the first run, `[5, 2, 3, 4]` and sort it using
  `insertionSort`. This becomes `[2, 3, 4, 5]`. We push its start
  index and length on the stack (`(0, 4)`).
* We next take `[9, 1, 6, 8]`, sort it to `[1, 6, 8, 9]`, and push
  `(4, 4)` on the stack.
* Finally, we only have `[10, 7]`. We sort this short run to `[7, 10]`
  and push `(6, 2)` on the stack.

Note all our sorting happens in-place, so by now the whole input became
`[2, 3, 4, 5, 1, 6, 8, 9, 7, 10]`. We then proceed to merge runs from
the top of the stack:

* First, we merge `[1, 6, 8, 9]` with `[7, 10`, which yields
  `[1, 6, 7, 8, 9, 10]`. We pop the two runs from the stack and push
  `(4, 6)`, the index and length of this new run.
* Next, we merge `[2, 3, 4, 5]` with `[1, 6, 7, 8, 9, 10]`, and update
  the stack accordingly. At this point, we only have 1 run on the
  stack (`[0, 10)`). We are done.

## Some Optimizations

So far, we haven't relied that much on the fact that our input might be
naturally partially sorted. Instead of simply calling `insertionSort` on
`minRun` elements, we can actually check whether elements are already
ordered. If they are, we don't need to do anything with them. Even
better, if the run of elements is longer than `minRun`, we keep going.

Elements might also come naturally sorted in descending order, while we
are sorting in ascending order. No problem: we can take a range of
elements coming in descending order and reverse it to produce a run in
ascending order. Let's call this function `countRunAndMakeAscending`:

``` python
def countRunAndMakeAscending(arr, lo, hi):
    runHi = lo + 1
    if runHi == hi:
        return 1

    if arr[lo] > arr[runHi]: # Descending run
        while runHi < hi and arr[runHi] < arr[runHi - 1]:
            runHi += 1
        reverseRange(arr, lo, runHi)
    else: # Ascending run
        while runHi < hi and arr[runHi] >= arr[runHi - 1]:
            runHi += 1

    return runHi - lo
```

We return the length of the run starting from `lo`, going to at most
`hi - 1`. If we have a natural descending run, we reverse the range
before returning. Here is `reverseRange`:

``` python
def reverseRange(arr, lo, hi):
    hi -= 1
    while lo < hi:
        arr[lo], arr[hi] = arr[hi], arr[lo]
        lo += 1
        hi -= 1
```

We can't get rid of sorting though: we might have worst-case scenario
cases with very small runs, in which case we still need a range of at
least `minRun` size. Based on the result of `countRunAndMarkAscending`,
if it is smaller than `minRun`, we will "force" a few more elements
into the run and sort it. Our new implementation looks like this:

``` python
def sort(arr): 
    lo, hi = 0, len(arr) 
    stack = []
    nRemaining = hi
    minRun = MIN_MERGE

    while nRemaining > 0:
        runLen = countRunAndMakeAscending(arr, lo, hi)

        if runLen < minRun:
            force = min(nRemaining, minRun)
            insertionSort(arr, lo, lo + force)
            runLen = force

        stack.append((lo, runLen))

        lo += runLen
        nRemaining -= runLen

    while len(stack) > 1:
        base2, len2 = stack.pop()
        base1, len1 = stack.pop()
        merge(arr, base1, base2, base2 + len2)
        stack.append((base1, len1 + len2))
```

Highlighting the changed part:

``` python
runLen = countRunAndMakeAscending(arr, lo, hi)

if runLen < minRun:
    force = min(nRemaining, minRun)
    insertionSort(arr, lo, lo + force)
    runLen = force
```

Instead of simply taking the next `minRun` elements, we try to find a
run. If the run we find is smaller than `minRun`, we force it to be
`minRun` by insertion-sorting into it more elements. If it is larger
than or equal to `minRun` on the other hand, we don't have to do any
sorting.

It gets better: now we know after calling `countRunAndMakeAscending`
that the range `[lo, lo + runLen)` is already sorted. We can hint this
to our sorting function and have it start sorting only from
`lo + runLen`. We can update `insertionSort` to take a hint of where to
start from:

``` python
def insertionSort(arr, lo, hi, start): 
    if start == lo:
        start += 1

    while start < hi:
        elem = arr[start]
        j = start - 1

        while elem < arr[j] and j >= lo:
        j -= 1

        arr.pop(start)
        arr.insert(j + 1, elem)

        start += 1
```

This version is very similar to our previous one. Instead of using a
local `i` variable to iterate over the range `[lo + 1, hi)`, we just use
`start`. If `start` is `lo`, we increment it before the loop (just like
we used to initialize `i` to `lo + 1`).

We can now pass this hint in from our main function:

``` python
while nRemaining > 0:
    runLen = countRunAndMakeAscending(arr, lo, hi)

    if runLen < minRun:
        force = min(nRemaining, minRun)
        insertionSort(arr, lo, lo + force, lo + runLen)
        runLen = force
```

At this point, we're starting to get a lot of value from naturally
sorted runs: we either don't do any sorting, or just sort at most
`minRun - runLen` elements into the range.

A further optimization for sorting: we can replace insertion sort with
binary sort. Binary sort works much like insertion sort, but instead of
checking where element `i` fits into `[lo, i)` by comparing it with
`i - 1`, then `i - 2` and so on, it relies on the fact that `[lo, i)` is
already sorted and performs a binary search to find the right spot. Here
is an implementation, which also takes a `start` hint:

``` python
def binarySort(arr, lo, hi, start):
    if start == lo:
        start += 1

    while start < hi:
        pivot = arr[start]
        left, right = lo, start

        while left < right:
            mid = (left + right) // 2

            if pivot < arr[mid]:
                right = mid
            else:
                left = mid + 1

        arr.pop(start)
        arr.insert(left, pivot)

        start += 1
```

Our main function now looks like this:

``` python
def sort(arr): 
    lo, hi = 0, len(arr) 
    stack = []
    nRemaining = hi
    minRun = MIN_MERGE

    while nRemaining > 0:
        runLen = countRunAndMakeAscending(arr, lo, hi)

        if runLen < minRun:
            force = min(nRemaining, minRun)
            binarySort(arr, lo, lo + force, lo + runLen)
            runLen = force

        stack.append((lo, runLen))

        lo += runLen
        nRemaining -= runLen

    while len(stack) > 1:
        base2, len2 = stack.pop()
        base1, len1 = stack.pop()
        merge(arr, base1, base2, base2 + len2)
        stack.append((base1, len1 + len2))
```

## Balanced Merges

Another key optimization of Timsort is trying as much as possible to
merge runs of balanced sizes. The closer the size, the better average
performance as a combination of additional space required and number of
operations.

So far we just pushed everything onto a stack, then merged the top 2
elements of the stack until we ended up with a single run. We actually
want to do something a bit different: we want our stack to maintain a
couple of invariants:

1.  `stack[i - 1][1] > stack[i][1] + stack[i + 1][1]` - the length of a
    run needs to be larger than the sum of the lengths of the following
    runs.
2.  `stack[i][1] > stack[i + 1][1]` - the length of a run needs to be
    larger than the following run.

When pushing a new index and run length tuple onto the stack, we check
if the invariant still holds. If it doesn't, we merge `stack[i]` with
the smallest of `stack[i - 1]`, `stack[i + 1]` and recheck. We continue
merging until the invariants are re-established. Let's call this
function `mergeCollapse`:

``` python
def mergeCollapse(arr, stack):
    while len(stack) > 1:
        n = len(stack) - 2
        if (n > 0 and stack[n - 1][1] <= stack[n][1] + stack[n + 1][1]) or \
        (n > 1 and stack[n - 2][1] <= stack[n][1] + stack[n - 1][1]):
        if stack[n - 1][1] < stack[n + 1][1]:
            n -= 1
        elif n < 0 or stack[n][1] > stack[n + 1][1]:
            break

        mergeAt(arr, stack, n)
```

We start from the top of the stack - 2. If `n > 0` and the invariant
doesn't hold for `stack[n - 1]`, `stack[n]`, and `stack[n + 1]` or if
`n > 1` and the invariant doesn't hold for `stack[n - 2]`,
`stack[n - 1]` and `stack[n]`, we need to merge. We decide whether we
want to merge `stack[n]` with `stack[n + 1]` or `stack[n - 1]` with
`stack[n]` depending on which one is smallest (if `stack[n - 1]` is
smaller, then we decrement `n` to trigger the merge at `n - 1`.

If the invariant holds, we check for the other invariant:
`stack[n][1] > stack[n + 1][1]`. If this second invariant holds, we're
done and we can break out of the loop (we do the same if we ran out of
elements). If not, we trigger a merge by calling `mergeAt` and repeat
until we either merge everything or the invariant is reestablished.

We start by checking only the top few elements of the stack, since we
expect the rest of the stack to hold the invariants. We only call this
function when we push a new run on the stack, in which case we need to
ensure we merge as needed.

Let's take a look at `mergeAt`. This function simply merges the runs at
positions `n` and `n + 1` on the stack:

``` python
def mergeAt(arr, stack, i):
    assert i == len(stack) - 2 or i == len(stack) - 3

    base1, len1 = stack[i]
    base2, len2 = stack[i + 1]

    stack[i] = (base1, len1 + len2)

    if i == len(stack) - 3:
        stack[i + 1] = stack[i + 2]
    stack.pop()

    merge(arr, base1, base2, base2 + len2)
```

Remember we only ever merge either the second from top and top runs or
the third from top and second from top runs. So `i` should be either
`len(stack) - 2` or `len(stack) - 3`. We get the first element and run
length for the two runs and update the stack: `stack[i]` starts at the
same position but will now have the length of both unmerged runs. If we
are merging `stack[-3]` with `stack[-2]`, we need to copy `stack[-1]`
(top of the stack) to `stack[-2]` (second to top). Finally, we pop the
top of the stack. At this point, the stack is updated. We call `merge`
on the two runs to update `arr` too.

We can now maintain a healthy balance for merges. Remember, the whole
reason for this is to aim to always merge runs similar in size.

Of course, once we are done pushing everything on the stack, we still
need to force merging to finish our sort. We'll do this with
`mergeForceCollapse`:

``` python
def mergeForceCollapse(arr, stack):
    while len(stack) > 1:
        n = len(stack) - 2
        if n > 0 and stack[n - 1][1] < stack[n + 1][1]:
            n -= 1

        mergeAt(arr, stack, n)
```

This function again merges the second from the top run with the smallest
of third from the top or top. It continues until all runs are merged
into one. Our updates `sort` looks like this:

``` python
def sort(arr): 
    lo, hi = 0, len(arr) 
    stack = []
    nRemaining = hi
    minRun = MIN_MERGE

    while nRemaining > 0:
        runLen = countRunAndMakeAscending(arr, lo, hi)

        if runLen < minRun:
            force = min(nRemaining, minRun)
            binarySort(arr, lo, lo + force, lo + runLen)
            runLen = force

        stack.append((lo, runLen))
        mergeCollapse(arr, stack)

        lo += runLen
        nRemaining -= runLen

    mergeForceCollapse(arr, stack)
```

Instead of pushing everything onto the stack and merging everything at
the end, we now call `mergeCollapse` after each push to keep the runs
balanced. At the end, we call `mergeForceCollapse` to force-merge the
stack.

## Run Lengths

We used a constant minimum run length so far, but mentioned earlier that
it is in fact determined as a function of the size of the input. We will
determine this with `minRunLength`:

``` python
def minRunLength(n): 
    r = 0
    while n >= MIN_MERGE: 
        r |= n & 1
        n >>= 1
    return n + r 
```

This function takes the length of the input and does the following:

* If `n` is smaller than `MIN_MERGE`, returns `n` - the input size is
  too small to use complicated optimizations on.
* If `n` is a power of 2, the algorithm will return `MIN_MERGE / 2`.
  Note: `MIN_MERGE` is also a power of 2. In our initial sketch we set
  it to 4, but in practice this is usually 32 or 64.
* Otherwise return a number `k` between `MIN_MERGE / 2` and
  `MIN_MERGE` so that `n / k` is close to but strictly less than a
  power of 2.

It does this by shifting `n` one bit to the right until it is less than
`MIN_MERGE`. In case any shifted bit is 1, it means `n` is not a power
of 2. In that case, we set `r` to 1 and return `n + 1`.

The reason we do all of this work is to again strive to keep merges
balanced. If we get an input like 2048 and our MIN_MERGE is 64, we get
back 32. That means that, if we don't have any great runs in our input,
we end up with 64 runs, each of length 32. We saw in the previous
section how we balance the stack. Consider we're pushing these runs
onto the stack:

* We push the run `(0, 32)` on the stack (first 32 elements).
* We push the run `(32, 32)` on the stack (next 32 elements).
* This triggers a merge since the run `(0, 32)` is not greater than
  the run `(32, 32)`. The stack becomes `(0, 64)`.
* We push the run `(64, 32)` on the stack (next 32 elements).
* We push the run `(96, 32)` on the stack (next 32 elements).
* This again triggers a merge, since the length of the run
  `(0, 64)` (64) is not greater than the length of the next two runs,
  both of which are 32. The run `(64, 32)` gets merged with the
  smaller run, `(96, 32)`. The stack becomes `[(0, 64), (64, 64)]`.
* The second invariant no longer holds: the first run is not longer
  than then next one. Another merged is triggered and the stack
  becomes `[(0, 128)]`.

This goes on in the same fashion, and all merges end up being perfectly
balanced. This works great for powers of 2.

Now let's consider another case: what if the input is 2112? If we would
still use 32 as our minimum run length, we would get 66 runs of length
32. The first 64 will trigger perfectly balanced merges as before, but
then we end up with the stack `[(0, 2048), (2048, 32), (2080, 32)]`.
This collapses to `[(0, 2048), (2048, 64)]`, triggering a completely
unbalanced merge (2048 on one side and 64 on the other).

To keep things balanced, if our input is not a power of 2, we pick a
minimum run length that is close to but strictly less than a power of 2.
Let's update our `MIN_MERGE` to be 32, and update our `sort` to call
`minRunLength` instead of automatically setting it to `MIN_MERGE`.
We'll throw in another quick optimization: if the whole input is
smaller than `MIN_MERGE`, don't even bother with the whole thing: find
a starting run then binary sort the rest, without any merging.

``` python
MIN_MERGE = 32

def sort(arr):
    lo, hi = 0, len(arr)
    stack = []
    nRemaining = hi
    if nRemaining < MIN_MERGE:
        initRunLen = countRunAndMakeAscending(arr, lo, hi)
        binarySort(arr, lo, hi, lo + initRunLen)
        return
    minRun = minRunLength(len(arr))
    while nRemaining > 0:
        runLen = countRunAndMakeAscending(arr, lo, hi)
        if runLen < minRun:
            force = min(nRemaining, minRun)
            binarySort(arr, lo, lo + force, lo + runLen)
            runLen = force
        stack.append((lo, runLen))
        mergeCollapse(arr, stack)
        lo += runLen
        nRemaining -= runLen
    mergeForceCollapse(arr, stack)
```

## Optimized Merging

We can optimize merging further. Our initial implementation of `merge`
simply copied the first run into a buffer, then performed the merge. We
can do better than that.

What if the second run is smaller? Maybe we'd prefer always merging the
smaller run into the larger one. Let's look at an optimized version of
merge. First, we'll replace `merge` with two functions, `mergeLo` and
`mergeHi`. `mergeLo` will copy elements from the first run into the
temporary buffer, while `mergeHi` will copy elements from the second
run. Our original `merge` becomes `mergeLo`, and we can add a `mergeHi`:

``` python
def mergeHi(arr, lo, mid, hi):
    t = arr[mid:hi]
    i, j, k = hi - 1, mid - 1, hi - mid - 1
    while k >= 0 and j >= lo:
        if t[k] > arr[j]:
            arr[i] = t[k]
            k -= 1
        else:
            arr[i] = arr[j]
            j -= 1
        i -= 1

    if k >= 0:
        arr[lo:i + 1] = t[0:k + 1]
```

This is very similar with `merge`, except it copies the second (`mid` to
`hi`) run into a temporary buffer and traverses the runs and the buffer
from end to start.

When we trigger the merge, another optimization we can do is check
elements from the first run and see if they are smaller than the first
element in the second run. While they are smaller, we can simply ignore
them when merging - they are already in position. We do this by taking
the first element of the second run and seeing where it would fit in the
first run.

Similarly, elements from the end of the second run which are greater
than the last element in the first run are already in place. We don't
need to touch them. We take the last element of the first run and check
where it would fit in the first run.

We can use binary search for this. Note that we need two version in
order to maintain the stable property of the sort: a `searchLeft`, which
returns the first index where a new element should be inserted, and a
`searchRight`, which returns the last index. For example, if we have a
run like `[1, 2, 5, 5, 5, 5, 7, 8]` and we are looking for where to
insert another `5`, it really depends where it comes from. If it comes
from the run before this one, we need the left-most spot (before the
first `5` in the run). On the other hand, if it comes from the run after
this one, we need to place it after the last `5`. That ensures that the
relative order of elements is preserved. Here is an implementation for
`searchLeft` and `searchRight`:

``` python
def searchLeft(key, arr, base, len):
    left, right = base, base + len
    while left < right:
        mid = left + (right - left) // 2
        if key > arr[mid]:
            left = mid + 1
        else:
            right = mid

    return left - base

def searchRight(key, arr, base, len):
    left, right = base, len

    while left < right:
        mid = left + (right - left) // 2
        if key < arr[mid]:
            right = mid
        else:
            left = mid + 1

    return left - base
```

Both functions return the offset from `base` where `key` should be
inserted.

We can now update our `mergeAt` function with the new capabilities:

``` python
def mergeAt(arr, stack, i):
    base1, len1 = stack[i]
    base2, len2 = stack[i + 1]

    stack[i] = (base1, len1 + len2)
    if i == len(stack) - 3:
        stack[i + 1] = stack[i + 2]
    stack.pop()

    k = searchRight(arr[base2], arr, base1, len1)
    base1 += k
    len1 -= k
    if len1 == 0:
        return

    len2 = searchLeft(arr[base1 + len1 - 1], arr, base2, len2)
    if len2 == 0:
        return

    if len1 > len2:
        mergeLo(arr, base1, base2, base2 + len2)
    else:
        mergeHi(arr, base1, base2, base2 + len2)
```

The first part stays the same: we get `base1`, `len1`, `base2`, and
`len2` and update the stack. Next, instead of merging right away, we
first search for where the first element of the second run would go into
the first run. We know the elements in `[base1, k)` won't move, so we
can remove them from the merge by moving `base1` to the right `k`
elements (we also need to update `len1`). Similarly, we search for where
the last element of the first run (`arr[base1 + len1 - 1]`) would fit
into the second run. We know all elements beyond that are already in
place, so we update `len2` to be this offset.

In case either of the searches exhausts a run, we simply return.
Otherwise, depending on which run is longer, we call `mergeLo` or
`mergeHi`.

## Galloping

But wait, there's more! Binary search always performs `log(len + 1)`
comparisons where `len` is the length of the array we are searching for
regardless of where our element belongs. Galloping attempts to find the
spot faster.

Galloping starts by comparing the element we are searching for in array
`A` with `A[0]`, `A[1]`, `A[3]`, ... `A[i^2 - 1]`. With these
comparisons, we will end up finding a range between some
`A[(k - 1)^2 - 1]` and `A[k^2 - 1]` that would contain the element we
are searching for. We then run a binary search only within that
interval.

There are some tradeoffs here: on large datasets or purely random data,
binary search performs better. But on inputs which contain natural runs,
galloping tends to find things faster. Galloping also performs better
when we expect to find the interval early on. Let's look at an
implementation of `gallopLeft` as an alternative to `searchLeft`:

``` python
def gallopLeft(key, arr, base, len, hint):
    lastOfs, ofs = 0, 1

    if key > arr[base + hint]:
        maxOfs = len - hint
        while ofs < maxOfs and key > arr[base + hint + ofs]:
            lastOfs = ofs
            ofs = (ofs << 1) + 1

        if ofs > maxOfs:
            ofs = maxOfs

        lastOfs += hint
        ofs += hint
    else: # key <= arr[base + hint]
        maxOfs = hint + 1
        while ofs < maxOfs and key <= arr[base + hint - ofs]:
            lastOfs = ofs
            ofs = (ofs << 1) + 1

        if ofs > maxOfs:
            ofs = maxOfs

        lastOfs, ofs = hint - ofs, hint - lastOfs

    # arr[base + lastOfs] < key <= arr[base + ofs]
    lastOfs += 1
    while lastOfs < ofs:
        mid = lastOfs + (ofs - lastOfs) // 2
        if key > arr[base + mid]:
            lastOfs = mid + 1
        else:
            ofs = mid
    return ofs
```

We start by initializing 2 offsets: `lastOfs` and `ofs` to represent the
offsets between which we expect to find our key. Note the function also
takes a hint, so callers can provide a tentative starting place.

Let's go over the parts of this function:

``` python
if key > arr[base + hint]:
    maxOfs = len - hint
    while ofs < maxOfs and key > arr[base + hint + ofs]:
        lastOfs = ofs
        ofs = (ofs << 1) + 1

    if ofs > maxOfs:
        ofs = maxOfs

    lastOfs += hint
    ofs += hint
```

We first find the two offsets. If the key we are searching for is
greater than (right of) our starting element (`arr[base + hint]`), then
our maximum possible offset is `len - hint`. While `ofs` is hasn't
overflowed and the key is still larger than `arr[base + hint + ofs]`, we
keep updating `ofs` to be the next power of 2 minus 1. We keep track of
the previous offset in `lastOfs`. Once we're done, we add `hint` to
both offsets (we do that because we add `hint` to all indices in our
loop, but not to `ofs` since we keep it a power of 2 minus 1). If
`key > arr[base + hint]` is not true, in other words, our key is left of
our starting element:

``` python
else: # key <= arr[base + hint]
    maxOfs = hint + 1
    while ofs < maxOfs and key <= arr[base + hint - ofs]:
        lastOfs = ofs
        ofs = (ofs << 1) + 1

    if ofs > maxOfs:
        ofs = maxOfs

    lastOfs, ofs = hint - ofs, hint - lastOfs
```

In this case, our maximum possible offset is `hint + 1`. We gallop
again, but now we are looking at elements left of our starting point,
`arr[base + hint - ofs]` where `ofs` keeps increasing. Once we find the
range, we update our offsets: `lastOfs` becomes `hint - ofs` and `ofs`
becomes `hint - lastOfs`. The `hint -` part is again because that is
what we actually used as indices. The swap is because we were moving
left, and we need `lastOfs` to be the one on the left, `ofs` the one on
the right.

We now identified the range within which we'll find our key, between
`arr[base + lastOfs]` and `arr[base + ofs]`. The last part of the
function is just a binary search within this interval.

The `gallopRight` function is very similar to `gallopLeft`:

``` python
def gallopRight(key, arr, base, len, hint):
    ofs, lastOfs = 1, 0

    if key < arr[base + hint]:
        maxOfs = hint + 1
        while ofs < maxOfs and key < arr[base + hint - ofs]:
            lastOfs = ofs
            ofs = (ofs << 1) + 1

        if ofs > maxOfs:
            ofs = maxOfs
        lastOfs, ofs = hint - ofs, hint - lastOfs
    else:
        maxOfs = len - hint
        while ofs < maxOfs and key >= arr[base + hint + ofs]:
            lastOfs = ofs
            ofs = (ofs << 1) + 1

        if ofs > maxOfs:
            ofs = maxOfs

        lastOfs += hint;
        ofs += hint;

    lastOfs += 1
    while lastOfs < ofs:
        mid = lastOfs + ((ofs - lastOfs) // 2)
        if key < arr[base + mid]:
            ofs = mid
        else:
            lastOfs = mid + 1
    return ofs
```

We won't cover this in details: the difference is here, like with
`searchRight`, we want to find the rightmost index where key belongs
instead of the leftmost one, so the algorithm changes accordingly.

The very neat thing about galloping is that its use isn't limited to
only when we set up the merge. We can also gallop while merging. Let's
go over `mergeLo` example, since `mergeHi` is a mirror of this.

In `mergeLo`, we first copy all elements from the first run to a buffer,
then we iterate over the array and at each position we copy either an
element from the buffer or one from the second run, depending on which
one is smaller. While we do this, we can keep track of how many times
the buffer or the second run "won". If one of these wins consistently,
we can assume it will keep winning for a while longer.

For example, if we merge `[5, 6, 7, 8, 9]` with `[0, 1, 2, 3, 4]`, we
initialize the buffer with `[5, 6, 7, 8, 9]`, but for the next 5
comparisons, the second run wins (`0 < 5`, `1 < 5` ...). Now imagine
much longer runs. Instead of comparing all elements one by one, we
switch to a galloping mode:

We find the last spot where the next element of the second run would fit
into the buffer, and immediately copy the preceding elements of the
buffer into the array. For example, if our buffer is
`[12, 13, 14, 15, 17]` and the element we are considering from the
second run is `[16]`, we know we can copy `[12, 13, 14, 15]` into the
array. Similarly, we find the first spot the next element in the buffer
would fit into the remaining second run, and copy elements before that
from the second run to their position. The galloping mode aims to reduce
the number of comparisons and bulk copy data when possible (using a
`memcpy` equivalent where available). While galloping, we still keep
track of how many elements we were able to skip comparing individually.
If this falls below the galloping threshold, we switch back to
"regular" mode. Here is an updated `mergeLo` implementation:

``` python
MIN_GALLOP = 7
minGallop = MIN_GALLOP

def mergeLo(arr, lo, mid, hi):
    t = arr[lo:mid]
    i, j, k = lo, mid, 0
    global minGallop
    done = False

    while not done:
        count1, count2 = 0, 0
        while (count1 | count2) < minGallop:
            if t[k] < arr[j]:
                arr[i] = t[k]
                count1 += 1
                count2 = 0
                k += 1
            else:
                arr[i] = arr[j]
                count1 = 0
                count2 += 1
                j += 1
            i += 1

            if k == mid - lo or j == hi:
                done = True
                break

        if done:
            break

        while count1 >= MIN_GALLOP or count2 >= MIN_GALLOP:
            count1 = gallopRight(arr[j], t, k, mid - lo - k, 0)
            if count1 != 0:
                arr[i:i + count1] = t[k:k + count1]
                i += count1
                k += count1
                if k == mid - lo:
                    done = True
                    break

            arr[i] = arr[j]
            i += 1
            j += 1
            if j == hi:
                done = True
                break

            count2 = gallopLeft(t[k], arr, j, hi - j, 0)
            if count2 != 0:
                arr[i:i + count2] = arr[j:j + count2]
                i += count2
                j += count2
                if j == hi:
                    done = True
                    break

            arr[i] = t[k]
            i += 1
            k += 1
            if k == mid - lo:
                done = True
                break

            minGallop -= 1

        if minGallop < 0:
            minGallop = 0
        minGallop += 2

    if k < mid - lo:
        arr[i:hi] = t[k:mid - lo]
```

We introduced a new `MIN_GALLOP` constant which is the threshold after
we want to start galloping. We also maintain a `minGallop` variable
across merges.

We have a couple of nested `while` loops, but the idea is pretty
straightforward. The first nested `while` does the normal merge but now
keeps track of how many times in the row did we end up picking an
element from the buffer:

``` python
count1, count2 = 0, 0
while (count1 | count2) < minGallop:
    if t[k] < arr[j]:
        arr[i] = t[k]
        count1 += 1
        count2 = 0
        k += 1
    else:
        arr[i] = arr[j]
        count1 = 0
        count2 += 1
        j += 1
    i += 1

    if k == mid - lo or j == hi:
        done = True
        break

if done:
    break
```

Whenever we increment one counter, we set the other to 0, so at any
point, at most one of them is different than 0. We can exit the while
loop in two ways: either one of the counters reaches the gallop
threshold, or we run out of elements in one of the arrays.

If we ran out of elements we are done, so we break out of the outer
loop. Otherwise we are in gallop mode:

``` python
while count1 >= MIN_GALLOP or count2 >= MIN_GALLOP:
    count1 = gallopRight(arr[j], t, k, mid - lo - k, 0)
    if count1 != 0:
        arr[i:i + count1] = t[k:k + count1]
        i += count1
        k += count1
        if k == mid - lo:
            done = True
            break

    arr[i] = arr[j]
    i += 1
    j += 1
    if j == hi:
        done = True
        break

    count2 = gallopLeft(t[k], arr, j, hi - j, 0)
    if count2 != 0:
        arr[i:i + count2] = arr[j:j + count2]
        i += count2
        j += count2
        if j == hi:
            done = True
            break

    arr[i] = t[k]
    i += 1
    k += 1
    if k == mid - lo:
        done = True
        break

    minGallop -= 1

if minGallop < 0:
    minGallop = 0
minGallop += 2
```

We first try to find where the next element in the second run would fit
into the buffer. That becomes our `count1`. If we get an offset greater
than 0, we can bulk copy the previous elements from the buffer
(`[k, k + count1)`) to the range `[i, i + count1)` and increment both
`k` and `i` by `count1`. Once we're done, we know for sure we need to
copy the next element from the second run (`a[j]`), so we do that.

We then do the opposite: gallop left to find where the next element from
the buffer would fit into the second run. That becomes our `count2` and
if it is greater than 0, we bulk copy elements from the second run. Once
we're done, we again now that the next element to copy is at `t[k]`, so
we do that.

This loop repeats while either `count1` or `count2` is greater than
`MIN_GALLOP`. If galloping works, we also update `minGallop` to favor
future galloping. Each time we iterate, we decrement `minGallop`. Once
we're out of the loop, if it is due to both `count1` and `count2` being
smaller than `MIN_GALLOP`, we again adjust `minGallop` - first, if it
became negative, we make it 0. We then add 2 to penalize galloping
because our last iteration didn't meet `MIN_GALLOP`. As a reminder,
`minGallop` is used as the threshold in the first loop. These tweaks to
`minGallop` aim to optimize, depending on the data, when to enter gallop
mode and when to keep merging in normal mode.

`minGallop` state should be maintained across multiple merges, and only
reset when we start a new sort - so we would make
`minGallop = MIN_GALLOP` in our main `sort` function, but otherwise rely
on the same value we are updating in `minGallop` for subsequent calls of
`mergeLo` and `mergeHi`. We made `minGallop` a global to keep the code
(relatively) simpler. To avoid globals, we should either put all
functions in a class and have minGallop be a member, or pass it through
as an argument through all functions that need it.

Finally, we copy the remaining elements in the buffer, if any:

``` python
if k < mid - lo:
    arr[i:hi] = t[k:mid - lo]
```

We also have the mirrored `mergeHi` version:

``` python
def mergeHi(arr, lo, mid, hi):
    t = arr[mid:hi]
    i, j, k = hi - 1, mid - 1, hi - mid - 1
    global minGallop
    done = False

    while not done:
        count1, count2 = 0, 0
        while (count1 | count2) < minGallop:
            if t[k] > arr[j]:
                arr[i] = t[k]
                count1 += 1
                count2 = 0
                k -= 1
            else:
                arr[i] = arr[j]
                count1 = 0
                count2 += 1
                j -= 1
            i -= 1

            if k == -1 or j == lo - 1:
                done = True
                break

        if done:
            break

        while count1 >= MIN_GALLOP or count2 >= MIN_GALLOP:
            count1 = j - lo + 1 - gallopRight(t[k], arr, lo, j - lo + 1, j - lo)
            if count1 != 0:
                arr[i - count1 + 1:i + 1] = arr[j - count1 + 1:j + 1]
                i -= count1
                j -= count1
                if j == lo - 1:
                    done = True
                    break

            arr[i] = t[k]
            i -= 1
            k -= 1

            if k == -1:
                done = True
                break

            count2 = k + 1 - gallopLeft(arr[j], t, 0, k + 1, k)
            if count2 != 0:
                arr[i - count2 + 1:i + 1] = t[k - count2 + 1:k + 1]
                i -= count2
                k -= count2
                if k == -1:
                    done = True
                    break

            arr[i] = arr[j]
            i -= 1
            j -= 1
            if j == lo - 1:
                done = True
                break

            minGallop -= 1

        if minGallop < 0:
            minGallop = 0
        minGallop += 2

    if k >= 0:
        arr[lo:i + 1] = t[0:k + 1]
```

This is very similar to the previous one, so I won't break it into
pieces and explain, just note that since we are starting from the end of
the range and we go backwards, we use closed ranges: `i`, `j`, and `k`
always point to the last element of the range, not the one past the
last.

## Summary

This is a very efficient sorting algorithm which relies on observed
properties of datasets in the real world. Quick recap:

* Depending on the size of the input, we determine a good size for
  runs, so we can get balanced merges.
* We traverse the array and identify runs. If the run is descending,
  we reverse it. If we don't get enough elements in a run to
  hopefully get balanced merges, we extend the run by adding more
  elements and sorting them using binary sort.
* We push runs on a stack which maintains a couple of invariants to,
  again, keep merges balanced: the second to top run of the stack must
  be longer than the top run and the third to top run must be longer
  than the sum of the second and top runs.
* If an invariant is violated, we start merging until we reestablish
  it. We merge the second from the top run with the shortest of third
  from top or top (again aiming for balanced overall merging). Merges
  always merge consecutive runs.
* Merge is optimized such that we first identify elements at the
  beginning of the first run and the end of the second run which are
  already in place, and we skip them.
* Next, depending on which of the runs is larger, we merge either from
  left or from right.
* Merge happens in two modes: we compare and merge normally, until we
  see one of the two runs we're merging consistently gets picked.
  Once we pass a certain threshold, we switch to galloping mode.
* Galloping aims to provide better performance than binary sort on
  smaller datasets, where we expect to find the position we're
  searching for earlier rather than later in the search. Galloping
  tries to find a `k` such that the position we looking for is within
  `A[(k - 1)^2]` and `A[k^2]`, then performs a binary search in the
  interval.
* Merging in galloping mode tries to find a range of elements in the
  run that tends to win. This range can be bulk-copied in the merge
  portion of the array more efficiently and skipping extra
  comparisons.
* If galloping becomes less effective, merge switches back to normal
  mode.
* Another heuristic keeps track of how well galloping mode performs
  and either encourages or discourages entering galloping mode again.
  This is persisted across multiple merges in a single sort.

## Thoughts

Is this sorting algorithm *beautiful*? Maybe not from a purely
syntactical/readability perspective. Compare it with the recursive
quicksort implementation in Haskell:

``` hs
quicksort :: (Ord a) => [a] -> [a]  
quicksort [] = []  
quicksort (x:xs) =   
    let smallerSorted = quicksort [a | a <- xs, a <= x]  
        biggerSorted = quicksort [a | a <- xs, a > x]  
    in  smallerSorted ++ [x] ++ biggerSorted  
```

Timsort is not a succinct algorithm. There are special cases,
optimizations for left to right and right to left cases, galloping,
which tries to beat binary search in some situations, multi-mode merges
and so on.

That said, everything in it has one purpose: sort real world data
efficiently. I find it beautiful for the amount of research that went
into it, the major insight that real world data is usually partially
sorted, and for how it adapts to various patterns in the data to improve
efficiency.

Most real world software looks more like Timsort than the Haskell
quicksort above. And while there is, unfortunately, way too much
accidental complexity in the world of software, there is a limit to how
much we can simplify before we can no longer model reality, or operate
efficiently. And, ultimately, that is what matters.

## References

The final version of the code in this blog post is in [this GitHub
gist](https://gist.github.com/vladris/13bf84513e76b75a60b0eb761207541e)
(be advised: implementation might be buggy).

Tim Peters has a very detailed explanation of the algorithm and all
optimizations in the Python codebase as
[listsort.txt](https://hg.python.org/cpython/file/tip/Objects/listsort.txt).
I do recommend reading this as it talks about all the research and
benchmarks that went into developing Timsort.

The C implementation of Timsort in the Python codebase is
[listobject.c](https://hg.python.org/cpython/file/tip/Objects/listobject.c).

The Python implementation relies on a lot of Python runtime constructs,
so it might be harder to read. My implementation is derived from the
OpenJDK implementation which I found very readable. That one is [here on
GitHub](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/TimSort.java).
