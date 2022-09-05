# Composable Generators

One of the most exciting features coming to C++ are coroutines. In this
post, I will give a quick overview of how they are used today in C# to
support generators and go over a few possible ways to bring the
composability of Linq to the C++ world.

## Generators in C#

I will not go into the nitty-gritty details of coroutines, but in short,
they are resumable functions -- functions that can be suspended/resumed.
Coroutines enable lazy evaluation, with two major applications: easy to
read multi-threading (with async/await syntax in C#) and generators
(with yield return syntax in C#). In this post, I will focus on
generators and how they compose.

I will start with a C# example since Linq is, in my opinion, the golden
standard for creating a processing pipeline, at least for non-functional
languages. Linq is implemented as a set of extension methods for
`IEnumerable`, and enables some very readable chaining of operations.
For example, let's get the first 100 natural numbers, filter out the
odds, then square the remaining list.

The wrong way of doing this would be something like:

``` c#
static IEnumerable<int> GetNumbers()
{
    for (int i = 0; i < 100; i++)
        if (i % 2 == 0)
            yield return i * i;
}

static void Main(string[] args)
{
    var result = GetNumbers();

    foreach (var item in result)
        Console.Write(item + " ");
}
```

The main problem with the above is that all the logic is inlined into
`GetNumbers`, so things don't decompose very well -- for example what if
we also want a function that squares the odd numbers? We would either
duplicate the looping and squaring logic, or make the predicate we use
to filter out things an input to the function. Same goes for the
iterating logic and for the squaring. Luckily, we have Linq, which does
just that:

``` c#
static void Main(string[] args)
{
    var result =
        Enumerable.Range(0, 100).
        Where(x => x % 2 == 0).
        Select(x => x * x);

    foreach (var item in result)
        Console.Write(item + " ");
}
```

To illustrate the magic of generators, instead of relying on
`Enumerable.Range`, let's introduce a function that generates numbers
forever:

``` c#
static IEnumerable<int> Count()
{
    for (int i = 0; ; i++)
        yield return i;
}
```

Our code would then become:

``` c#
static void Main(string[] args)
{
    var result =
        Count().
        Take(100).
        Where(x => x % 2 == 0).
        Select(x => x * x);

    foreach (var item in result)
        Console.Write(item + " ");
}
```

While not strictly necessary in this particular case, infinite
generators cannot exist without lazy evaluation, a feature of many
functional languages. Lazy evaluation has some very practical
applications as it allows processing of data as it becomes available,
instead of waiting for everything to be ready before moving on to the
next step. While the 100 natural numbers example might not sound so
useful, imagine rendering frames in a streaming video as they arrive
over the network. Linq is great because it provides a clean separation
between the generic algorithms (`Where`, `Select` etc.) and the
problem-specific operations which are passed in as arguments. Linq
operations also compose well, so they can be chained together to form
pipelines.

## Generators in C++

While coroutines haven't made it into the C++17 standard itself, they
are coming as a technical specification, with MSVC already supporting
them (code samples below compile with VS 2015 Update 3). The main syntax
additions are the new `co_await`, `co_return`, and `co_yield` keywords.
The first two are used for creating and awaiting tasks (which I won't
cover in this post), while `co_yield` is used in generators.

Here is a lazy counter in C++:

``` c++
auto count_to(int n) -> std::experimental::generator<int>
{
    for (int i = 0; i < n; i++)
        co_yield i;
}

int main()
{
    for (int i : count_to(100))
        std::cout << i << " ";
}
```

Note the return type of `count_to` is a `generator<int>` (currently in
the experimental namespace). `generator<T>` is the type implicitly
created by the compiler when encountering a `co_yield`. Also worth
noting that range-based for loops work over generators, as they expose
`begin()` and `end()` methods. The type annotation for the `count_to`
return type above is not really needed, I added it just to clarify what
the complier will generate in this case.

`generator` itself is pretty bare-boned, it doesn't provide all the
algorithms that Linq adds to `IEnumerable`. So if we wanted to do
something like the above pipeline, we would need some algorithms. Here's
one way of implementing some of them:

``` c++
auto count()
{
    for (int i = 0; i < 100; i++)
        co_yield i;
}

template <typename T>
auto take_n(std::experimental::generator<T> gen, int n)
{
    int i = 0;
    for (auto&& item : gen)
        if (i++ < n)
            co_yield item;
        else
            return;
}

template <typename T, typename Predicate>
auto filter(std::experimental::generator<T> gen, Predicate pred)
{
    for (auto&& item : gen)
        if (pred(item))
            co_yield item;
}

template <typename T, typename BinaryOperation>
auto map(std::experimental::generator<T> gen, BinaryOperation op)
{
    for (auto&& item : gen)
        co_yield op(item);
}
```

Here I switched from Linq's `Select` and `Where` to the more commonly
used `map` and `filter`, but they effectively implement the same thing.
While this implementation is pretty-straight forward, it doesn't compose
well at all:

``` c++
int main()
{
    auto result =
        map(
            filter(
                take_n(count(), 100),
                [](int x){ return x % 2 == 0; }),
            [](int x){ return x * x;});

    for (auto&& item : result)
        std::cout << item << " ";
}
```

Definitely not like the nice chaining of Linq. So what gives? Why
doesn't generator come out-of-the-box with `take_n`, `map`, `filter` and
all the other useful algorithms? Well, according to the [single
responsibility
principle](https://en.wikipedia.org/wiki/Single_responsibility_principle),
these algorithms don't belong in `generator` -- `generator` encapsulates
the lazy evaluation of the coroutine, it wouldn't be the right place for
algorithms. It's also worth noting that Linq methods are not part of
`IEnumerable`, they are [extension
methods](https://msdn.microsoft.com/en-us/library/bb383977.aspx). C++
doesn't support extension methods, so we would need a slightly different
design to achieve better chaining.

## Decorator

The next idea comes from pure OOP - let's create a decorator over
`generator` that exposes these algorithms. First, let's declare our
decorator as `enumerable<T>` and change our algorithms to work with the
new type:

``` c++
template <typename T>
struct enumerable;

template <typename T>
auto take_n(enumerable<T> gen, int n) -> enumerable<T>
{
    int i = 0;
    for (auto&& item : gen)
        if (i++ < n)
            co_yield item;
        else
            return;
}

template <typename T, typename Predicate>
auto filter(enumerable<T> gen, Predicate pred) -> enumerable<T>
{
    for (auto&& item : gen)
        if (pred(item))
            co_yield item;
}

template <typename T, typename BinaryOperation>
auto map(enumerable<T> gen, BinaryOperation op) -> enumerable<T>
{
    for (auto&& item : gen)
        co_yield op(item);
}
```

The implementation looks pretty much like before, except that now we are
getting and returning `enumerable<T>` instead of `generator<T>`. In this
case the type annotation is mandatory, as by default the complier would
create a `generator<T>`.

We can then implement our enumerable to wrap a generator and expose
member functions which forward to the above algorithms:

``` c++
template <typename T>
struct enumerable
{
    // Needed by compiler to create enumerable from co_yield
    using promise_type = typename std::experimental::generator<T>::promise_type;

    enumerable(promise_type& promise) : _gen(promise) { }
    enumerable(enumerable<T>&& other) : _gen(std::move(other._gen)) { }

    enumerable(const enumerable<T>&) = delete;
    enumerable<T> &operator=(const enumerable<T> &) = delete;

    auto begin() { return _gen.begin(); }
    auto end() { return _gen.end(); }

    auto take_n(int n)
    {
        return ::take_n(std::move(*this), n);
    }

    template <typename Predicate>
    auto filter(Predicate pred)
    {
        return ::filter(std::move(*this), pred);
    }

    template <typename BinaryOperation>
    auto map(BinaryOperation op)
    {
        return ::map(std::move(*this), op);
    }

    std::experimental::generator<T> _gen;
};
```

A few things to note: we declare a `promise_type` and have a constructor
which takes a promise as an argument. This is required by the compiler
when creating the object on `co_yield`. We follow the same semantics as
generator, since that is what we are wrapping -- support only
move-constructor, no copy-constructor. All the member algorithms do a
destructive move on `*this`. This is intentional, as once we iterate
over the encapsulated generator, it is no longer valid. Since we don't
expose a copy-constructor, we move out of `*this` when passing the
generator to an algorithm. For completeness, we can also provide a
function which converts from a generator to an enumerable:

``` c++
template <typename T>
auto to_enumerable(std::experimental::generator<T> gen) -> enumerable<T>
{
    for (auto&& item : gen)
        co_yield item;
}
```

This works, and we can now compose algorithms by chaining the calls:

``` c++
int main()
{
    auto result =
        to_enumerable(count()).
        take_n(100).
        filter([](int x) { return x % 2 == 0; }).
        map([](int x) { return x * x; });

    for (auto&& item : result)
        std::cout << item << " ";
}
```

Still, it is not ideal -- first, we need to explicitly tell the compiler
everywhere to return our type with `co_yield` instead of the default
generator, and we need to handle conversions to and from the standard
library generator. The enumerable algorithms compose well, but we'll
have trouble composing with functions that work with generators. Also,
having a huge class consisting solely of algorithms is not the best
design, especially in a language where free functions are first class
citizens.

## Pipe Operator

An alternative approach, which the [Boost Ranges
library](http://www.boost.org/doc/libs/1_62_0/libs/range/doc/html/index.html)
takes, is to overload `|`, the "pipe" operator, so we can compose our
calls like this:

``` c++
int main()
{
    auto result =
        count() |
        take_n(100) |
        filter([](int x) { return x % 2 == 0; }) |
        map([](int x) { return x * x; });

    for (auto&& item : result)
        std::cout << item << " ";
}
```

One way we can get this working is to first create a type that wraps an
algorithm and an `operator|` implementation between a lhs `generator`
and a rhs of our type:

``` c++
template <typename Predicate>
struct filter_t
{
    filter_t(Predicate pred) : _pred(pred) { }

    template<typename T>
    auto operator()(std::experimental::generator<T> gen) const
    {
        for (auto&& item : gen)
            if (_pred(item))
                co_yield item;
    }

    const Predicate _pred;
};

template <typename T, typename Predicate>
auto operator|(
    std::experimental::generator<T> lhs,
    const filter_t<Predicate>& rhs)
{
    return rhs(std::move(lhs));
}
```

Here, `filter_t` holds the `Predicate` we want to use, and `operator|`
applies it on the given `generator`. This works, but we wouldn't be able
to instantiate `filter_t` with a lambda like in the above chaining
example without specifying the Predicate type in the call. If we want to
leverage type deduction, we can create a simple helper function that
creates a `filter_t` from a given argument:

``` c++
template <typename Predicate>
auto filter(Predicate pred)
{
    return filter_t<Predicate>(pred);
}
```

With this we can call `| filter(/* predicate */)` on a generator and get
back a filtered generator. Full implementation for `take_n`, `filter`
and `map` would be:

``` c++
struct take_n_t
{
    take_n_t(int n) : _n(n) { }

    template <typename T>
    auto operator()(std::experimental::generator<T> gen) const
    {
        int i = 0;
        for (auto&& item : gen)
            if (i++ < _n)
                co_yield item;
            else
                return;
    }

    const int _n;
};

template <typename T>
auto operator|(
    std::experimental::generator<T> lhs,
    const take_n_t& rhs)
{
    return rhs(std::move(lhs));
}

auto take_n(int n)
{
    return take_n_t(n);
}

template <typename Predicate>
struct filter_t
{
    filter_t(Predicate pred) : _pred(pred) { }

    template<typename T>
    auto operator()(std::experimental::generator<T> gen) const
    {
        for (auto&& item : gen)
            if (_pred(item))
                co_yield item;
    }

    const Predicate _pred;
};

template <typename T, typename Predicate>
auto operator|(
    std::experimental::generator<T> lhs,
    const filter_t<Predicate>& rhs)
{
    return rhs(std::move(lhs));
}

template <typename Predicate>
auto filter(Predicate pred)
{
    return filter_t<Predicate>(pred);
}

template <typename BinaryOperation>
struct map_t
{
    map_t(BinaryOperation op) : _op(op) { }

    template <typename T>
    auto operator()(std::experimental::generator<T> gen) const
    {
        for (auto&& item : gen)
            co_yield _op(item);
    }

    const BinaryOperation _op;
};

template <typename T, typename BinaryOperation>
auto operator|(
    std::experimental::generator<T> lhs,
    const map_t<BinaryOperation>& rhs)
{
    return rhs(std::move(lhs));
}

template <typename BinaryOperation>
auto map(BinaryOperation op)
{
    return map_t<BinaryOperation>(op);
}
```

With this approach, we can apply our algorithms over a generator without
having to introduce a different type. They also compose very nicely, the
only slightly odd thing being using the `|` operator (though as I
mentioned, there is a precedent for this in Boost and chances are it
might show up in other places in the future).

## Unified Call Syntax

One thing that would've made things even easier but unfortunately was
not approved for C++17 is [unified call
syntax](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4165.pdf).
At a high level, unified call syntax would make the compiler try to
resolve `x.f()` to `f(x)` if `decltype(x)` doesn't have an `f()` member
function but there is a free function `f(decltype(x))`. Similarly, if no
`f(decltype(x))` exists but `decltype(x)` has a member function `f()`,
`f(x)` would resolve to the member function call `x.f()`.

If it's not obvious, unified call syntax would allow us to easily create
extension methods. We would be able to revert our algorithm code to the
first version:

``` c++
template <typename T>
auto take_n(std::experimental::generator<T> gen, int n)
{
    int i = 0;
    for (auto&& item : gen)
        if (i++ < n)
            co_yield item;
        else
            return;
}

template <typename T, typename Predicate>
auto filter(std::experimental::generator<T> gen, Predicate pred)
{
    for (auto&& item : gen)
        if (pred(item))
            co_yield item;
}

template <typename T, typename BinaryOperation>
auto map(std::experimental::generator<T> gen, BinaryOperation op)
{
    for (auto&& item : gen)
        co_yield op(item);
}
```

But now this becomes very composable as calling `take_n`, `filter` or
`map` on a generator would resolve to the free functions if the
`generator` itself does not have them as members:

``` c++
int main()
{
    auto result =
        count().
        take_n(100).
        filter([](int x) { return x % 2 == 0; }).
        map([](int x) { return x * x; });

    for (auto&& item : result)
        std::cout << item << " ";
}
```

The above currently does not compile but it should (disclaimer: slight
tweaks might be required) if unified call syntax becomes part of the
standard.

## In Summary

We went over a couple of alternatives to implement some common
algorithms over C++ generators with a focus on composability:

* Stand-alone functions are simple but don't compose very well.
* Using a decorator works, but is not ideal from a design point of
  view and not very idiomatic.
* Using the pipe operator for chaining and helper types for the
  algorithms is the best approach today.
* Unified call syntax would simplify things a lot, enabling a
  mechanism to implement these algorithms as extension methods.
