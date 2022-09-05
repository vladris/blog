# Clean Code: Types

I recently revived my Clean Code tech talk which I put together a couple
of years ago and with which I started this blog: [Clean Code - Part
1](https://vladris.com/blog/2016/01/04/clean-code-part-1.html) and
[Clean Code - Part
2](https://vladris.com/blog/2016/01/07/clean-code-part-2.html). I took
the opportunity to completely revamp the talk and ended up with 3 parts:
*Algorithms*, *Types*, and *State*. The *Algorithms* is mostly covered
by the [Fibonacci](https://vladris.com/blog/2018/02/11/fibonacci.html)
post, so in this post we will talk about *Types*.

## Mars Climate Orbiter

The Mars Climate Orbiter crashed and disintegrated in the Mars
atmosphere because a component developed by Lockheed provided momentum
measured in pound-force seconds, while another component developed by
NASA expected momentum as Newton seconds.

We can image the component developed by NASA being something like this:

``` c++
// Will not disintegrate as long as momentum >= 2 N s
void trajectory_correction(double momentum)
{
    if (momentum < 2 /* N s */)
    {
        disintegrate();
    }
    /* ... */
}
```

We can also imagine the Lockheed component calling into the above with:

``` c++
void main()
{
    trajectory_correction(1.5 /* lbf s */);
}
```

A pound-force second (lbfs) is about 4.448222 Newton seconds (Ns). So
from Lockheed's perspective, passing in 1.5 lbfs to
`trajectory_correction` should be just fine: 1.5 lbfs is about 6.672333
Ns, way above the 2 Ns threshold.

The problem is the interpretation of the data. The NASA component ends
up comparing lbfs to Ns without conversion, misinterpreting the lbfs
input as Ns. Since 1.5 is less than 2, the orbiter disintegrates. This
is a known anti-pattern called "primitive obsession".

## Primitive Obsession

Primitive obsession happens when we use a primitive data type to
represent a value in the problem's domain and causes situations like
the above. Representing zip codes as numbers, telephone numbers as
strings, Ns and lbfs as `double` are all examples of this.

A more type safe solution would have defined a simple `Ns` type:

``` c++
struct Ns
{
    double value;
};

bool operator<(const Ns& a, const Ns& b)
{
    return a.value < b.value;
}
```

We can similarly define a simple `lbfs` type:

``` c++
struct lbfs
{
    double value;
};

bool operator<(const lbfs& a, const lbfs& b)
{
    return a.value < b.value;
}
```

Now we can implement a type safe `trajectory_correction`:

``` c++
// Will not disintegrate as long as momentum >= 2 N s
void trajectory_correction(Ns momentum)
{
    if (momentum < Ns{ 2 })
    {
        disintegrate();
    }
    /* ... */
}
```

Calling this with `lbfs` as below fails to compile as the types are
incompatible:

``` c++
void main()
{
    trajectory_correction(lbfs{ 1.5 });
}
```

Note how the meaning of the values, which used to be specified in
comments (`2 /* Ns */`, `/* lbfs */`) gets pulled into the type system
and expressed in code (`Ns{ 2 }`, `lbfs{ 1.5 }`).

We can, of course, provide casting from `lbfs` to `Ns` as an explicit
operator:

``` c++
struct lbfs
{
    double value;

    explicit operator Ns()
    {
        return value * 4.448222;
    }
};
```

Equipped with this, we can call `trajectory_correction` via a static
cast:

``` c++
void main()
{
    trajectory_correction(static_cast<Ns>(lbfs{ 1.5 }));
}
```

This does the right thing of multiplying by the ratio. The cast can also
be made implicit (by using the `implicit` keyword instead), in which
case it is applied automatically. As a rule of thumb, it's best to
follow the Zen of Python:

> Explicit is better than implicit

The moral of the story is that nowadays we have very sophisticated type
checkers but we do need to provide them enough information to catch this
type of errors. That information comes from declaring types to represent
our problem domain. [^1]

## State Space

Bad things happen when our programs end up in a *bad state*. Types help
us narrow down the possibility of such bad states. One way to think
about this is to look at types as sets of possible values. For example
`bool` is the set `{true, false}` where a variable of the type can be
one of the two values. Similarly, `uint32_t` is the set
`{0 ... 4294967295}`. Looking at types like this, we can define the
*state space* of our program as the product of the types of all live
variables at a given point in time.

If we have a `bool` and an `uint32_t`, our state space is
`{true, false} X {0 ... 4294967295}`. This simply means that the two
variables can be in any of their possible states and since we have two
of them, our program can be in any of their combined states.

This gets more interesting when we look at functions that initialize
values:

``` c++
bool get_momentum(Ns& momentum)
{
    if (!some_condition()) return false;

    momentum = Ns{ 3 };

    return true;
}
```

In the above example we take a `Ns` by reference and initialize it if
some condition is met. The function returns `true` if the value was
properly initialized. If the function cannot, for whatever reason, set
the value, it returns `false`.

Looking at this from the state space lens, our state space is the
product `bool X Ns`. If the function returns `true`, then `momentum` was
set and is in any one of the possible `Ns` values. The problem is that
if the function returns `false`, then `momentum` was not set. It is
still in any one of the possible `Ns` values, but it is not a valid
value. Often times we have bugs where we accidentally propagate such
invalid state:

``` c++
void example()
{
    Ns momenum;

    get_momentum(momentum);

    trajectory_correction(momentum);
}
```

What we should have done instead is:

``` c++
void example()
{
    Ns momentum;

    if (get_momentum(momentum))
    {
        trajectory_correction(momentum);
    }
}
```

There is a better way though, where this can be enforced:

``` c++
std::optional<Ns> get_momentum()
{
    if (!some_condition()) return std::nullopt;

    return std::make_optional(Ns{ 3 });
}
```

Using an `optional`, this version of the function has a significantly
smaller state space: instead of `bool X Ns`, we have `Ns + 1`. The
function either returns a valid `Ns` value or `nullopt` to denote the
absence of a value. Now it becomes impossible to have an invalid `Ns`
that gets propagated throughout the system. We can also no longer
*forget* to check the return value as an `optional<Ns>` is not
implicitly convertible to an `Ns` - we need to explicitly unpack it:

``` c++
void example()
{
    auto maybeMomentum = get_momentum();

    if (maybeMomentum)
    {
        trajectory_correction(*maybeMomentum);
    }
}
```

In general, we want our functions to return **result or error** not
**result and error**. This way we eliminate the states in which we have
an error but also an invalid result which might make its way in further
computation.

From this point of view, throwing exceptions is OK as this follows the
same pattern: a function either returns a result **or** throws an
exception.

## RAII

RAII stands for *Resource Acquisition Is Initialization* but has more to
do with releasing resources. The name originated from C++ but the
pattern can be implemented in any language (see, for example, .NET's
`IDisposable`). RAII ensures automatic cleanup of resources.

What are resources? A few examples: heap memory, database connections,
OS handles. In general, a resource is something we acquire from the
outside world and we need to release when it is no longer needed. That
means executing some form of free, delete, close etc. on the resource.

Since these resources are external, they are not directly expressed into
our type system. For example if we allocate some heap memory, we get a
pointer on which we have to call `delete`:

``` c++
struct Foo {};

void example()
{
    Foo* foo = new Foo();

    /* Use foo */

    delete foo;
}
```

But what happens if we forget or something prevents us from calling
`delete`?

``` c++
void example()
{
    Foo* foo = new Foo();

    throw std::exception();

    delete foo;
}
```

In this case we no longer call `delete` and we leak the resource. In
general, we don't want to perform such manual cleanup. For heap memory,
we actually have `unique_ptr` to help us manage it:

``` c++
void example()
{
    auto foo = std::make_unique<Foo>();

    throw std::exception();
}
```

The `unique_ptr` is a stack object so whenever it goes out of scope
(when the function throws or during stack unwinding if an exception was
thrown) its destructor gets called. It's destructor implements the call
to `delete`. This way, we no longer have to manually manage the memory
resource - we hand it off to a wrapper which owns it and handles
releasing it.

Similar wrappers exist or can be created for any of the other resources
(for example a Windows OS `HANDLE` can be wrapped in a type where its
destructor would call `CloseHandle`.

The key takeaway is never to do manual resource cleanup - either use an
existing wrapper or, if none exists for your particular scenario,
implement one.

## Summary

This post started with a famous example of why typing is important, and
covered three important aspects of leveraging types to write safer code:

* Declaring and using stronger types (as opposed to primitive
  obsession).
* Reducing state space, returning result or error instead of result
  and error.
* RAII and automatic resource management.

Types are great tools for implementing safer, reusable code.

[^1]: There is a great series of posts on Fluent C++ on [Strong
    Typing](https://www.fluentcpp.com/category/strong-types/).
