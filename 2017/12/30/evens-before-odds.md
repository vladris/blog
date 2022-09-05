# Evens before Odds

One of my go-to interview questions goes like this:

> Given an array of numbers, make it so the even numbers come before the
> odd ones.
>
> For example, for `{ 1, 2, 3, 4, 5, 6, 7, 8 }`, a possible output would
> be `{ 8, 2, 6, 4, 5, 3, 7, 1 }`.

This is not a trick question by any means, it is a straightforward
problem with a couple of straightforward solutions. Note the *possible
output* wording and the fact that evens and odds in the output do not
preserve the relative order they had in the input.

## A Space-Inefficient Solution

An easy solution is to traverse the input once and store all even
numbers encountered during the traversal into a separate array, then
traverse it again and do the same for the odd numbers:

``` c++
auto evens_before_odds(const std::vector<int>& numbers)
{
    std::vector<int> result;
    result.reserve(numbers.size());

    for (int i : numbers)
        if (i % 2 == 0)
            result.push_back(i);

    for (int i : numbers)
        if (i % 2 != 0)
            result.push_back(i);

    return result;
}
```

This solves the problem in `O(n)` linear time (two traversals of the
input array) and `O(n)` linear space - result is as large as input, so
additional space required grows linearly with the size of the input.

There are more efficient way of doing this in linear time and constant
space.

## Two Algorithms

There are a couple of ways we can solve this. One algorithm goes like
this:

> Find the first odd number. Stop if we reached the end of the array.
>
> Find the first event number after that odd number. Stop if we reached
> the end of the array.
>
> Swap them.
>
> Repeat.

Our loop invariant is that all numbers before the first odd number
(which we update during each iteration) are already in the right place.
With each iteration, we find another even number that appears after the
first odd number so we swap them, putting the even in the right place.
We stop when we run out of numbers to swap, either odd or even. An
implementation of this algorithm looks like this:

``` c++
void evens_before_odds(std::vector<int>& numbers)
{
    int i = 0, j, n = numbers.size();

    for (;;)
    {
        // Find first odd number
        while (i != n && numbers[i] % 2 == 0)
            ++i;

        // If we reached the end we're done
        if (i == n)
            return;

        // Find the first even number after the first odd one
        j = i + 1;
        while (j != n && numbers[j] % 2 != 0)
            ++j;

        // If we reached the end we're done
        if (j == n)
            return;

        // Swap and continue
        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}
```

While the algorithm is fairly straight-forward, the devil is in the
details - we need to perform multiple checks to make sure we don't run
off the end of the array. While interviewing, I've seen many bugs come
up due to missing some of these checks.

An interesting observation we can make is that once we found the first
pair of odd and even numbers, after we swap them, the new first odd
number is right after the even we just swapped, so we can hoist the
first while statement out of the main loop - we only need to find the
first odd once, then we just increment after each swap:

``` c++
void evens_before_odds(std::vector<int>& numbers)
{
    int i = 0, j, n = numbers.size();

    // Find the first odd number
    while (i != n && numbers[i] % 2 == 0)
        ++i;

    // If we reached the end we’re done
    if (i == n)
        return;

    // Start after the first odd and until we reach the end
    for (int j = i + 1; j != n; ++j)
    {
        // If it’s an even number
        if (numbers[j] % 2 == 0)
        {
            // Swap with the first odd
            std::swap(numbers[i], numbers[j]);
            // Increment first odd position
            ++i;
        }
    }
}
```

Another algorithm goes like this:

> Find the first odd number.
>
> From the back, find the last even number.
>
> Stop if the first odd number appears after the last even number.
>
> Swap and repeat.

Our loop invariant is that all numbers before the first odd and all
numbers after the last even are already in place. With each iteration,
we move the first odd and last even. We stop when the first odd appears
after the last even, which means all evens appear before the odds. Here
is a possible implementation:

``` c++
void evens_before_odds(std::vector<int>& numbers)
{
    int i = 0, j = numbers.size();

    for (;;)
    {
        // Find the first odd number
        while (i != j && numbers[i] % 2 == 0)
            ++i;

        // If the first odd occurs after the last even, stop
        if (i == j)
            return;

        // Find the last even number
        --j;
        while (i != j && numbers[j] % 2 != 0)
            --j;

        // If the first odd occurs after the last even, stop
        if (i == j)
            return;

        // Swap and continue
        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}
```

Both of the above algorithms solve the problem in linear time and
constant space.

### Test Cases

Some interesting test cases to validate the implementations:

* Our example input `{1, 2, 3, 4, 5, 6, 7, 8 }`.
* An empty vector `{ }`.
* A vector with a single even number `{ 2 }`.
* A vector with a single odd number `{ 1 }`.
* A vector consisting of all even number `{ 2, 4, 6 }`.
* A vector consisting of all odd numbers `{ 1, 3, 5 }`.

### Follow Up: Odds before Evens

My follow up question is

> What if we also want the ability to put odd numbers before even ones?
> How would we extend our code?

An answer I'm **not** looking for is *we copy/paste the function, rename
it to* `odds_before_evens` *and update the checks*.

A clever answer (which I'm also not looking for) is *we provide an*
`odds_before_evens` *which internally calls* `evens_before_odds` *, then
reverses the output*:

``` c++
void odds_before_evens(std::vector<int>& numbers)
{
    evens_before_odds(numbers);
    std::reverse(numbers.begin(), numbers.end());
}
```

A common answer is *we add a flag*:

``` c++
void arrange_numbers(std::vector<int>& numbers, bool evensFirst)
{
    int i = 0, j = numbers.size();

    for (;;)
    {
        while (i != j && ((evensFirst && numbers[i] % 2 == 0)
            || (!evensFirst && numbers[i] %2 != 0)))
            ++i;

        if (i == j)
            return;

        --j;
        while (i != j && ((evensFirst && numbers[j] % 2 != 0)
            || (!evensFirst && numbers[j] % 2 == 0)))
            --j;

        if (i == j)
            return;

        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}
```

This kind of works, but the condition becomes very complicated.

### Follow Up: Primes before Non-Primes

What if we also want to move prime numbers before non-prime numbers,
given some `bool is_prime(int)` primality-testing function?

We can keep adding flags and extending the `if` conditions:

``` c++
enum class Arrangement
{
    EvensBeforeOdds,
    OddsBeforeEvens,
    PrimesBeforeNonPrimes,
};

void arrange_numbers(std::vector<int>& numbers, Arrangement arrangement)
{
    int i = 0, j = numbers.size();

    for (;;)
    {
        while (i != j && ((arrangement == Arrangement::EvensBeforeOdds && numbers[i] % 2 == 0)
            || (arrangement == Arrangement::OddsBeforeEvens && numbers[i] %2 != 0)
            || (arrangement == Arrangement::PrimesBeforeNonPrimes && is_prime(numbers[i]))))
            ++i;

        if (i == j)
            return;

        --j;
        while (i != j && ((arrangement == Arrangement::EvensBeforeOdds && numbers[j] % 2 != 0)
            || (arrangement == Arrangement::OddsBeforeEvens && numbers[j] % 2 == 0)
            || (arrangement == Arrangement::PrimesBeforeNonPrimes && !is_prime(numbers[j]))))
            --j;

        if (i == j)
            return;

        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}
```

This doesn't scale very well though. What we actually want to do here is
abstract away the predicate based on which we move elements around:

``` c++
template <typename Pred>
void arrange_numbers(std::vector<int>& numbers, Pred pred)
{
    int i = 0, j = numbers.size();

    for (;;)
    {
        while (i != j && pred(numbers[i]))
            ++i;

        if (i == j)
            return;

        --j;
        while (i != j && !pred(numbers[j]))
            --j;

        if (i == j)
            return;

        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}

void evens_before_odds(std::vector<int>& numbers)
{
    arrange_numbers(numbers, [](int i) { return i % 2 == 0; });
}

void odds_before_evens(std::vector<int>& numbers)
{
    arrange_numbers(numbers, [](int i) { return i % 2 != 0; });
}

void primes_before_non_primes(std::vector<int>& numbers)
{
    arrange_numbers(numbers, is_prime);
}
```

Note the algorithm remains the same: we have the exact same steps and
loop invariants, but we can parameterize the condition. With this
abstraction, the code actually becomes smaller and more readable.

This is about as far as I can get during an interview.

## Partition

This is actually a well-known algorithm called a *partitioning
algorithm*. A partitioning algorithm moves elements that satisfy a
predicate before elements that don't satisfy it. Let's start with the
above implementation:

``` c++
template <typename Pred>
void partition(std::vector<int>& numbers, Pred pred)
{
    int i = 0, j = numbers.size();

    for (;;)
    {
        while (i != j && pred(numbers[i]))
            ++i;

        if (i == j)
            return;

        --j;
        while (i != j && !pred(numbers[j]))
            --j;

        if (i == j)
            return;

        std::swap(numbers[i], numbers[j]);
        ++i;
    }
}
```

This works for vectors, but what if we want to partition a doubly-linked
list? Can we abstract away the data structure we are partitioning? The
answer is *yes*. We can use iterators to access the data structure:

``` c++
template <typename It, typename Pred>
void partition(It first, It last, Pred pred)
{
    for (;;)
    {
        while (first != last && pred(*first))
            ++first;

        if (first == last)
            return;

        --last;
        while (first != last && !pred(*last))
            --last;

        if (first == last)
            return;

        std::swap(*first, *last);
        ++first;
    }
}
```

The implementation is virtually the same. We get rid of `i` and `j`, as
we are using the iterators provided as arguments for traversal. The
implementation does not increase in complexity, but is now usable beyond
vectors. For example we can now partition a C-style array:

``` c++
void evens_before_odds(int arr[], int n)
{
    partition(arr, arr + n, [](int i) { return i % 2 == 0; });
}
```

### Useful Return

A useful return for our algorithm is the *partition point* - the
position of the first element that does not satisfy our predicate. We
have this implicitly and callers might be interested in it. To avoid
making callers have to recompute it, we should return it:

``` c++
template <typename It, typename Pred>
auto partition(It first, It last, Pred pred)
{
    for (;;)
    {
        while (first != last && pred(*first))
            ++first;

        if (first == last)
            return first;

        --last;
        while (first != last && !pred(*last))
            --last;

        if (first == last)
            return first;

        std::swap(*first, *last);
        ++first;
    }
}
```

For example, `partition` is a key ingredient in quicksort:

``` c++
template <typename It, typename Comp>
void quick_sort(It first, It last, Comp comp)
{
    // Stop if we have no elements or one element
    auto dist = std::distance(first, last);
    if (dist < 2) return;

    // Swap pivot with last element
    auto pivot = first + dist / 2;
    std::iter_swap(pivot, --last);

    // Partition around pivot
    auto p = partition(first, last, [&](auto&& i) {
        return comp(i, *last);
    });

    // Move pivot back in place
    std::iter_swap(p, last);

    // Recursively sort left and right sides of the pivot
    quick_sort(first, p, comp);
    quick_sort(p + 1, ++last, comp);
}
```

### STL Implementations

The `partition` algorithm we ended up with is fairly efficient, but it's
worth taking a look at some of the highly-optimized STL implementations.
This is the MSVC STL implementation:

``` c++
template <typename It, typename Pred>
auto partition(It first, It last, Pred pred)
{
    for (;; ++first)
    {
        for (; first != last && pred(*first); ++first);
        if (first == last) break;

        for (; first != --last && !pred(*last););
        if (first == last) break;

        iter_swap(first, last);
    }

    return first;
}
```

Note this performs the least possible amount of operations. It also
seems to favor `for` loops. Contrast this with the LLVM libc++
implementation, which seems to favor `while` loops:

``` c++
template <typename It, typename Pred>
auto partition(It first, It last, Pred pred)
{
    while (true)
    {
        while (true)
        {
            if (first == last) return first;
            if (!pred(*first)) break;
            ++first;
        }
        do
        {
            if (first == --last) return first;
        } while (!pred(*last));
        swap(*first, *last);
        ++first;
    }
}
```

### Iterator Requirements and Complexity

We focused on the second algorithm presented, which finds the first odd,
last even, and swaps them. We had another algorithm which was looking
for *the first even after the first odd* during each iteration. Let's
provide a generic implementation for it too:

``` c++
template <typename It, typename Pred>
auto partition(It first, It last, Pred pred)
{
    while (first != last && pred(*first))
        ++first;

    if (first == last)
        return first;

    for (It next = std::next(first); next != last; ++next)
        if (pred(*next))
        {
            std::swap(*first, *next);
            ++first;
        }

    return first;
}
```

What is the difference?

The difference is that this algorithm only ever increments the
iterators. That means it only requires a `ForwardIterator`, as opposed
to the other algorithm, which finds the *last even* number starting from
the `last` iterator, which requires a `BidirectionalIterator`.

In other words, the algorithm requiring only a `ForwardIterator` works
on a singly-linked list (`forward_list`), while the other one can't (we
can only traverse a singly-linked list forward in `O(1)` time, not
backwards).

The MSVC STL implementation of the forward-iterator algorithm is:

``` c++
template<typename It, typename Pred>
auto partition(It first, It last, Pred pred)
{
    while (first != last && pred(*first))
        ++first;

    if (first == last)
        return first;

    for (It next = next(first); next != last; ++next)
        if (pred(*next))
            iter_swap(first++, next);

    return first;
}
```

The libc++ one is:

``` c++
template <typename It, tpyename Pred>
auto partition(It first, It last, Pred pred)
{
    while (true)
    {
        if (first == last)
            return first;
        if (!pred(*first))
            break;
        ++first;
    }
    for (It next = first; ++next != last;)
    {
        if (pred(*next))
        {
            swap(*first, *next);
            ++first;
        }
    }
    return first;
}
```

The reason both implementations are provided is that the
`ForwardIterator` version, while more generally applicable, is slightly
less efficient. The `BidirectionalIterator` version moves any element at
most once, and since the move is a swap, it means it performs at most
`N / 2` swaps where `N` is the number of elements. The `ForwardIterator`
version might perform more swaps, up to `N`. For example, for the input
`1 2 4`, during the first step, it would swap `1` with `2`, ending up
with `2 1 4`, then during the next step it would swap `1` with `4`,
ending up with `2 4 1`.

## In C#

Partitioning is not specific to the C++ language. The same
implementation can be used, for example, in C#, up to abstracting away
data structure traversal:

``` C#
public static class IListPartition
{
    public static int Partition<T>(this IList<T> self, Func<T, bool> pred)
    {
        int first = 0, last = self.Count;

        for (;;)
        {
            while (first != last && pred(self[first]))
                ++first;

            if (first == last)
                return first;

            --last;
            while (first != last && !pred(self[last]))
                --last;

            if (first == last)
                return first;

            var temp = self[first];
            self[first] = self[last];
            self[last] = temp;
            ++first;
        }
    }
}
```

The .NET `IEnumerator` does not allow us to mutate the data structure we
are enumerating over, so we cannot provide a generic `IEnumerable<T>`
partition algorithm that works in-place. Otherwise the implementation is
pretty much identical to the C++ one, as the algorithm is the same.

## Summary

* Moving even numbers before odd ones in a given array of numbers is
  an instance of partition.
* The algorithm can be generalized to work with an arbitrary
  predicate.
* The algorithm can be generalized to work across any data structure
  as long as it can be traversed with at least a `ForwardIterator`.
* A `BidirectionalIterator` version performs at most `N / 2` swaps
  (and `N` applications of the predicate).
* A `ForwardIterator` version performs at most `N` swaps (and `N`
  applications of the predicate).
* Both versions of the algorithm are part of the standard library
  (`std::partition` algorithm).
* The same algorithm can be implemented in other languages, as generic
  as the available abstractions allow.
