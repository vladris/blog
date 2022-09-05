# Notes on Types

## Type Systems

A good type system can eliminate entire categories of errors in a
program and simply make invalid code not compile. Before digging into
types, below are a few common distinctions between the type systems of
various programming languages:

### Dynamic Typing vs Static Typing

In a statically typed language, the types are determined at compile time
so if a function was declared as only accepting a certain type `T` but
we attempt to pass it an unrelated type `U`, the program is considered
invalid. This is invalid C++:

``` c++
void sqr(int x)
{
    return x * x;
}
...
sqr("not an int"); // does not compile
```

On the other hand, in a dynamically typed language, we do not perform
any compile time checks and, if the data we get is of an unexpected
type, we treat it as a runtime error. Below is a Python function that
squares a number:

``` python
def sqr(x):
    return x ** 2
...
sqr("not an int")
## runs but fails with TypeError: unsupported operand type(s)
## for ** or pow(): 'str' and 'int'
```

Some of the interesting features of dynamic languages are *duck typing*
("if it walks like a duck, and it quacks like a duck...") and *monkey
patching*. Duck typing means that accessing a member of an object works
as long as that object has such a member, regardless of the type of the
object. In Python:

``` python
class Foo:
    def func(self):
        print("foo")

class Bar:
    def func(self):
        print("bar")

[obj.func() for obj in [Foo(), Bar()]] # prints "foo" and "bar"
```

This can't work in a statically typed language where, at the bare
minimum, we would have to add some form of constraint for the types in
the list to ensure they contain a `func()` method we can call.

Monkey patching refers to the ability to change the structure of an
object at runtime. For example we can swap the `func` method from an
instance of `foo` with another function like this:

``` python
class Foo:
    def func(self):
        print("foo")

def func_bar():
    print("bar")

obj = Foo()
obj.func() # prints "foo"
obj.func = func_bar
obj.func() # prints "bar"
```

These are useful capabilities, but the tradeoff is a whole class of type
errors which a statically typed language would've caught.

As a side note, the fact that dynamic languages don't need to specify
types makes them more terse. That being said, the [Hindley-Milner
algorithm
W](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system) can
infer the types of a program in linear time with respect to the source
size. So while Python is starting to support type annotations for better
static analysis and TypeScript provides a way for writing type-safe
JavaScript, C++ has better and better type inference, while in Haskell
(which has one of the strongest static type systems) type annotations
are mostly optional.

### Strong Typing vs Weak Typing

At a high level, a strongly typed language does not implicitly convert
between unrelated types. This is good in most situations as implicit
conversions are often meaningless or have surprising effects - for
example, adding a number to a list of characters. This can either result
in runtime errors or garbage data. In contrast, a strongly typed
language will not accept code that attempts to do this.

In Python, which is strongly typed, this doesn't work:

``` python
foo = "foo" # foo is "foo"
foo = foo + " bar" # foo is "foo bar"
foo = foo + 5 # TypeError: Can't convert 'int' object to str implicitly
```

It works just fine in JavaScript though:

``` javascript
var foo = "foo"; // foo is "foo"
foo = foo + " bar"; // foo is "foo bar"
foo = foo + 5; // foo is "foo bar5"
```

Note type strength is not an either/or - C++, while considered strongly
typed, still allows several implicit casts between types (eg. pointer to
bool). Some languages are more strict about converting between types
implicitly, others less so.

### Dynamic Polymorphism vs Static Polymorphism

Another difference to note is between static and dynamic polymorphism.
Dynamic polymorphism happens at runtime, when calling a function on a
base type gets resolved to the actual function of the deriving type:

``` c++
struct base
{
    virtual void func() = 0;
};

struct foo : base
{
    void func() override
    {
        std::cout << "foo" << std::endl;
    }
};

struct bar : base
{
    void func() override
    {
        std::cout << "bar" << std::endl;
    }
};

void call_func(base& obj)
{
    obj.func();
}
...
call_func(foo{}); // prints "foo"
call_func(bar{}); // prints "bar"
```

In the above case, we effectively have a single function `call_func`
which takes a reference to a `base` struct. The compiler generates a
v-table for `struct base` and a call to `func()` on `base` involves a
v-table jump to the actual implementation of the function, which is
different between the inheriting types `foo` and `bar`.

Contrast this with the static alternative:

``` c++
struct foo
{
    void func()
    {
        std::cout << "foo" << std::endl;
    }
};

struct bar
{
    void func()
    {
        std::cout << "bar" << std::endl;
    }
};

template <typename T> void call_func(T& obj)
{
    obj.func();
}
...
call_func(foo{}); // prints "foo"
call_func(bar{}); // prints "bar"
```

In this case there is no relationship between `foo` and `bar` and no
v-table is needed. On the other hand, we no longer have a single
`call_func`, we have a templated function which is instantiated for both
`foo` and `bar` types. This is all done at compile-time, the advantage
being faster code, the drawback being compiler needs to be aware of all
the types involved - we can no longer "inject" types implementing an
interface at runtime. When calling `call_func`, we need to have both the
definition of the function and the declaration of the type we're
passing in visible.

## Types

During the rest of this post, I will talk about types in the context of
a statically and strongly typed language, with a focus on static
polymorphism. This pushes as much as possible of the type checking to
the compilation stage, so many of the runtime issues of less strict
languages become invalid syntax.

I will focus on C++ and cover some of the new C++17 feature which enable
or make some of these concepts easier to work with. That being said,
since this post focuses on types, I will also provide examples in
Haskell, as Haskell can express these concepts much more succinctly.

### Type Basics

Let's start with the definition of a type: *a type represents the set
of possible values*. For example, the C++ type `uint8_t` represents the
set of integers from 0 to 255. Effectively this means that a variable of
a given type can only have values from within that set.

### Interesting Types

Since we defined a type as a set of possible values, we can talk about
the cardinality of a type, in other words the number of values in the
set. Based on cardinality, there are a few interesting classes of types
to talk about:

### Empty Type

The first interesting type to talk about is the type that represents the
empty set, with `|T| = 0`.

In Haskell, this type is named `Void`. Since Haskell is a functional
language, all functions must return a value, so it does not make sense
to have a function that returns `Void` - the same way it doesn't make
sense to define a mathematical function with the empty set as its
codomain. We do have an `absurd` function though, which maps the empty
set to any value:

``` haskell
absurd :: Void -> a
```

This function cannot be called though.

In C++, the absence of a value is represented as the `void` type. Since
C++ is not purely functional, we can define functions that don't return
anything. We can even say that a function does not take any arguments by
putting a `void` between the parenthesis:

``` c++
void foo(void);
```

This is the equivalent of:

``` c++
void foo();
```

Note though that we cannot have a *real* argument of type `void`, that
is a compile error as it doesn't make any sense - we would be mandating
the function takes a value from the empty set. So we can say `foo(void)`
but not `foo(void arg)`, or even `foo(int arg, void)`.

### Unit Type

The next interesting class consists of types with cardinality 1. A type
`T` with `|T| = 1` is called a *unit* or *singleton type*. A variable of
such a type can only ever have a single possible value. In Haskell, the
anonymous representation is the empty tuple `()`. Here is an example of
a function that maps anything to this type:

``` haskell
unit :: a -> ()
unit _ = ()
```

Of course, we can declare our own singleton types. Below is a custom
`Singleton` type and an equivalent unit function:

``` haskell
data Singleton = Singleton
unit :: a -> Singleton
unit _ = Singleton
```

In C++, the anonymous representation of a singleton is an empty
`std::tuple`:

``` c++
template <typename T> std::tuple<> unit(T)
{
    return { };
}
```

As can be seen from the above, Haskell makes it easier to define a
function that takes an argument of any type, as it provides syntactic
sugar for type parameters (`a` in our example). In C++, the equivalent
declaration involves a template, but they boil down to the same thing.
The non-anonymous C++ representation is a struct which doesn't contain
anything. All instances of such a struct are equivalent:

``` c++
struct singleton { };

template <typename T> singleton unit(T)
{
    return { };
}
```

### Sum Types

Here, things get a bit more interesting: a `sum type` is a type which
can represent a value from any of the types it sums. So given type `A`
and type `B`, the type `S` summing up `A` and `B` is
`S = { i : i ∈ A U B }`. So a variable of type `S` could have any value
in `A` or any value in `B`. `S` is called a sum type because its
cardinality is the sum of the cardinalities of `A` and `B`,
`|S| = |A| + |B|`.

Sum types are great, because they allow us to build up more complex
types from simpler ones. Once we have unit types, we can build up more
complex types out of them by summing them. For example, a boolean type
which can be either `true` or `false` can be thought of as the sum of
the singleton `true` type and the singleton `false` type. In Haskell, a
boolean is defined as:

``` haskell
data Bool = True | False
```

Similarly, a `Weekday` type can be defined as:

``` haskell
data Weekday = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday
```

Theoretically, numerical types could also be defined as huge sum types
of every possible value they can represent. Of course, this is
impractical, but we can reason about them the same way we reason about
other sum types, we don't have to treat them as a special case.

In C++, an equivalent of the above is an `enum class`. `bool` is a
built-in type with special syntax, but we could define an equivalent as:

``` c++
enum class Boolean
{
    True,
    False
};
```

It's easy to see how a `Weekday` definition would look like. Things get
more interesting when we throw type parameters into the mix. In Haskell,
we have the `Either` type, which is declared as follows:

``` haskell
data Either a b = Left a | Right b
```

An instance of this could be either a `Left a`, where `a` is a type
itself, which means it can be any of the values of `a`, or it can be
`Right b`, with any of the values of `b`. In Haskell we use
pattern-matching to operate on such a type, so we can declare a simple
function that tells us whether we were given a `Left a` like this:

``` haskell
isLeft :: Either a b -> Bool
isLeft (Left a) = True
isLeft (Right b) = False
```

This might not look like much, but of course we can compose more complex
functions. For example, say we have a function `foo` that takes an `a`
and returns an `a`, a function `bar` that takes a `b` and returns a `b`.
We can then write a `transform` function which takes an `Either a b`
and, depending on the contained type, it applies the appropriate
function:

``` haskell
-- Implementation of foo and bar not provided in this example
foo :: a -> a
bar :: b -> b

transform :: Either a b -> Either a b
transform (Left a) = Left (foo a)
transform (Right b) = Right (bar b)
```

This is way beyond the capabilities of a C++ `enum class`. The old way
of implementing something like this in C++ was using a union and a tag
enum to keep track of which is the actual type we're working with:

``` c++
// Declaration of A and B not provided in this example
struct A;
struct B;

struct Either
{
    Either(A a)
    {
        ab.left = a;
        tag = tag::isA;
    }

    Either(B b)
    {
        ab.right = b;
        tag = tag::isB;
    }

    union
    {
        A left;
        B right;
    } ab;

    enum class tag
    {
        isA,
        isB
    } tag;
};
```

Our implementation of transform would look like this:

``` c++
// Implementation of A and B not provided in this example
A foo(A);
B bar(B);

Either transform(Either either)
{
    switch (either.tag)
    {
        case Either::tag::isA:
            return foo(either.ab.left);
        case Either::tag::isB:
            return bar(either.ab.right);
    }
}
```

Our `Either` type definition is obviously much more verbose than what we
had in Haskell, and it doesn't even support type parameters - at this
point it only works with `struct A` and `struct B`, while the Haskell
version works for any `a` and `b` types. The other major problem is
that, while unions provide efficient storage for different types (the
size of the union is the size of the maximum contained type), it is up
to the implementer to make sure we don't try to read an `A` as a `B` or
vice-versa. That means we need to keep our tag in sync with what we put
in the type and respect it when accessing the value of the union.

C++17 introduces a better, safer, parameterized type for this:
`std::variant`. Variant takes any number of types as template arguments
and stores an instance of any one of those types. Using variant, we can
re-write the above as:

``` c++
std::variant<A, B> transform(std::variant<A, B> either)
{
    return std::visit([](auto e&&) {
        if constexpr (std::is_same_v<decltype(e), A>)
            return foo(std::get<A>(e));
        else
            return bar(std::get<B>(e));
    }, either);
}
```

This is a lot of new syntax, so let's break it down:
`std::variant<A, B>` is the new C++17 sum type. In this case, we specify
it holds either `A` or `B` (but it can hold an arbitrary number of
types).

`std::visit` is a function that applies the visitor function given as
its first argument to the variants given as its subsequent arguments. In
our example, this effectively expands to applying the lambda to
`std::get<0>(either)` and `std::get<1>(either)`.

`if constexpr` is also a new C++17 construct which evaluates the if
expression at compile time and discards the else branch from the final
object code. So in this example, we determine at compile time whether
the type we are being called with is `A` or `B` and apply the correct
function based on that. Something very similar can be achieved with
templates and `enable_if`, but this syntax makes for more readable code.

Note that with this version we can simply prepend a
`template <typename A, typename B>` and make the whole function generic,
as in the Haskell example. It doesn't read as pretty (because we don't
have good pattern matching syntax in the language), but this is the new,
type safe way of implementing and working with sum types, which is a
major improvement.

### Product Types

With sum types out of the way, the remaining interesting category is
that of *product types*. Product types combine the values of several
other types into one. For types `A` and `B`, we have
`P = { (a, b) : a ∈ A, b ∈ B }`, so `|P| = |A| x |B|`.

In Haskell, the anonymous version of product types is represented by
tuples, while the named version is represented by records. An example of
a `perimeter` function which computes the perimeter of a rectangle
defined by two points, where each point is a tuple of numbers:

``` haskell
perimeter :: (Num n) => (n, n) -> (n, n) -> n
perimeter (x1, y1) (x2, y2) = 2 * (abs(x1 - x2) + abs(y1 - y2))
```

The named version would declare a `Point` type with `Int` coordinates
and use that instead:

``` haskell
data Point = Point { x, y :: Int }

perimeter :: Point -> Point -> Int
perimeter (Point x1 y1) (Point x2 y2) = 2 * (abs(x1 - x2) + abs(y1 - y2))
```

The C++ equivalents are `std::tuple` for anonymous product types and
`struct` for named types:

``` c++
// Anonymous
int perimeter(std::tuple<int, int> p1, std::tuple<int, int> p2)
{
    return 2 * ((abs(std::get<0>(p1) - std::get<0>(p2))
        + abs(std::get<1>(p1) - std::get<1>(p2)));
}

// Named
struct point
{
    int x;
    int y;
};

int perimeter(point p1, point p2)
{
    return 2 * (abs(p1.x - p2.x) + abs(p1.y - p2.y));
}
```

While sum types allow us to express values from multiple types into one,
product types allow us to express values from several types together.
Empty, unit, sum, and product types are the building blocks of a type
system.

### Bonus: Optional Types

An optional type is a type that can hold any of the values of another
type, or not hold any value, which is usually represented as a
singleton. So an optional is effectively a sum type between a given type
and a singleton representing "doesn't hold a value". In other words,
the cardinality of an optional for a type `T` is `|O| = |T| + 1`.

In Haskell, an optional is the famous `Maybe` type:

``` haskell
data Maybe a = Just a | Nothing
```

A function that operates on `Maybe` could say "only apply `foo` if the
optional contains an `a`":

``` haskell
-- Implementation of foo not provided in this example
foo :: a -> a

transform :: Maybe a -> Maybe a
transform (Just a) = Just (foo a)
transform Nothing = Nothing
```

The new C++17 equivalent is the `optional` type:

``` c++
// Implementation of foo not provided in this example
A foo(A);

std::optional<A> transform(std::optional<A> a)
{
    return a != nullopt ? foo(*a) : nullopt;
}
```

This might read a bit like the pointer implementation:

``` c++
A* transform(A* a)
{
    return a != nullptr ? foo(*a) : nullptr;
}
```

There is a key difference though: the type contained in the optional is
part of the object, so it is not allocated dynamically the way we would
allocate a pointer. `nullopt` is a helper object of the singleton type
`nullopt_t`.

Types are important because a big part of programming effectively
consits of designing and composing types. Having a good understanding of
the fundamentals leads to better, safer, and saner code.

## Summary

We started by outlining some of the basic principles of type systems:

* Static and dynamic typing.
* Weak and strong typing.
* Static and dynamic polymorphism.

Then we went over the building block types of a type system, with
Haskell and C++ examples:

* Empty types (cardinality 0).
* Unit types (cardinality 1).
* Sum types (`S` of `A` and `B` has cardinality `A + B`).
* Product types (`P` of `A` and `B` has cardinality `A x B`).
* Optional types (`O` of `A` has cardinality `A + 1`).

## Further Reading

Ben Dean had an excellent talk at CppCon this year, [Using Types
Effectively](https://www.youtube.com/watch?v=ojZbFIQSdl8). Another great
talk about type design from CppCon is [C++, Abstract Algebra and
Practical Applications](https://www.youtube.com/watch?v=632a-DMM5J0) by
Robert Ramey. And then there is Bartosz Milewski
[blog](https://bartoszmilewski.com/) about Haskell, C++, and category
theory.
