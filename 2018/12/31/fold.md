# Fold

Most object-oriented programming languages represent object instances in
memory in two separate chunks. One of them contains the
instance-specific state - the class attributes. The other one contains
the methods, which are actually shared across instances. Let's take as
an example a simple type with a couple of properties and a method:

``` c++
class Point {
public:
    double X;
    double Y; 

    double distanceToOrigin() const { /* ... */ }
};
```

If we take two distinct instances of `Point`, like `new Point(10, 5)`
and `new Point(20, 20)`, they have different `X` and `Y` coordinates, so
those need to be stored individually. On the other hand, the logic of
`distanceToOrigin` is the same for both objects. The method evaluates to
different results, because `this->X` and `this->Y` are different for the
two objects, but the code is the same.

Instead of storing separate copies of the code for each instance, the
method implementations are shared across objects. `distanceToOrigin` can
return different results for different objects because each invocation
gets a different `this` pointer to the state of each object. In fact,
under the hood, every class method gets an implicit `this` argument
which represents the object on which the method is invoked. Python makes
that explicit, requiring all class methods to implement a `self`
argument in order to reference instance-specific state.

There isn't much difference between a method like
`Point::distanceToOrigin` and a free function which takes a `Point` as
an argument:

``` c++
double distanceToOrigin(Point p)
{
    /* ... */
}
```

The only difference is that a class member gets access to privates,
while an outside function doesn't. Visibility aside, methods of a class
are an independent "chunk" shared across object instances and the
state of each object is another, separate "chunk".

An interpretation of this is that, even though classes contain methods,
object instances of a given type can still be thought of as pure state.
Our `new Point(10, 5)` is represented by a combination of the `Point`
class, which includes the method implementations and their memory layout
etc. and instance-specific state. The `Point` class includes the method
`distanceToOrigin` with an implicit `this` argument (and some additional
metadata).

With this view, a `Point` instance is a pair of values `X` and `Y`. Any
particular instance of `Point` is a member of the set
`{ (X, Y) : X ∈ double, Y ∈ double }`. The function `distanceToOrigin`
can be viewed as a function from this set (from the implicit `this`
argument) to a `double` value.

`disntaceToOrigin : { (X, Y) : X ∈ double, Y ∈ double } → double.`

## Operations and Closures

A function that takes two arguments of type `Point` is called a binary
operation on the set. We can represent it as a function
`f(Point, Point)` or, equivalently, as an operation `Point ⊙ Point`. If
the codomain of the function is also a `Point`, for example let's take a
function like `Point midPoint(Point x, Point y)`, which computes the
middle point between two given points, we call the operation *closed*
over the set.

The notion of closure in algebra is different from the notion closure in
computer science, which deals with context captured in lambdas. In this
case it simply means an operation that combines a number of elements of
a set into another element of the set.

Since we can view any type as a set, and any function taking two
arguments of a type and returning another instance of that type as a
closed binary operation on the set, we can start talking about algebraic
structures.

## Magmas

A magma is just a set with a closed operation, without any other
constraints imposed. If we have a magma, we can implement an algorithm
like `fold` which, given a set of values in the magma, combines them
into a single value.

For example we can fold three `Point` instances `a`, `b`, and `c` into a
single `Point` using the `midPoint` function like this:

``` c++
Point result = midPoint(midPoint(a, b), c);
```

We can sketch out a function that, given a set of objects of any type
`T` and an operation closed over `T`, produces a final value of type
`T`:

``` c++
template <typename It, typename Op,
    typename T=typename std::iterator_traits<It>::value_type>
T fold(It begin, It end, T init, Op op)
{
    T result = init;

    for (auto it = begin; it != end; ++it) {
        result = op(result, *it);
    }

    return result;
}
```

This function takes a pair of iterators which we use to traverse the set
of values of type `T`, an initial value of type `T`, and a binary
operation `Op`. We need an initial value of type `T` because if the set
is empty, we still need to return something from the function. That is,
if the `for` loop never executes, and we never apply `op`, we still need
a `T` to return. In this case we will simply return `init`.

Since a magma doesn't impose any constraints on the operation, the
order in which we combine elements might be important. For example, for
integers and the subtraction operation, if we start with the set of
number `1`, `2`, and `3`, and an initial value of `1`, we can fold from
left to right and get `((1 - 1) - 2) - 3`, or `-5`. On the other hand,
we can start from right to left, and fold `1 - (2 - (3 - 1))`, or `1`.
Let's call this verison `foldRight` and look at a possible
implementation:

``` c++
template <typename It, typename Op
    typename T=typename std::iterator_traits<It>::value_type>
T fold_right(It begin, It end, T init, Op op)
{
    T result = init;

    for (auto it = end; it != begin; ) {
        result = op(*(--it), result);
    }

    return result;
}
```

Note that for an operation like addition, this is not the case: if we
fold `1`, `2`, and `3`, with an initial value of `1`, we get `7`
regardless of the direction and whether we make the initial value a left
or a right operand. Let's zoom in on this.

## Semigroups

If we have a set with a closed operation that is also associative, we
have a *semigroup*. For an *associative* operation, it doesn't really
matter what order we apply multiple operations on. For example, for the
set of strings and the string concatenation operation, it doesn't
metter whether we append `foo` to `bar`, then the result to `baz` or
whether we concatenate `bar` with `baz` first, then prepend `foo` to it:

``` c++
("foo"s + "bar"s) + "baz"s == "foo"s + ("bar"s + "baz"s)
```

More formally, an opeartion is associative if, for any `a`, `b`, and
`c`, `(a ⊙ b) ⊙ c == a ⊙ (b ⊙ c)`. That being said, that's not enough
to make our right fold redundant:

``` c++
std::vector<std::string> elems = { "foo"s, "bar"s, "baz"s };

std::cout << fold(elems.begin(), elems.end(), "!"s,
    [](const std::string& x, const std::string& y) { 
        return x + y; 
    });
// Prints !foobarbaz

std::cout << fold_right(elems.begin(), elems.end(), "!"s,
    [](const std::string& x, const std::string& y) { 
        return x + y;
    });
// Prints foobarbaz!
```

It still matters whether our initial value is a left or right argument
to our operation. If for an operation we get the same result regardless
of how we arrange the operands, then we have a *commutative* operation.
Integer addition is an example of a commutative operation, where `1 + 2`
is the same as `2 + 1`. Note this is different than string
concatenation, where `"foo"s + "bar"s != "bar"s + "foo"s`. A semigroup
with a commutative operation is called an *abelian semigroup*. Here we
finally no longer need to distinguish between `fold` and `fold_right` as
they both produce the same result.

Let's also look at that mandatory initial value.

## Monoids

The reason we require an initial value is that we need to return
something in case our set is empty, and we don't always have a good
default. That's not always the case. There are operations for which we
have such a default, called the *identity* of the opeartion. This value,
combined with any other value using the operation, leaves the other
value unchanged. For string concatenation, the identity is the empty
string. For addition, the identity is `0`. In general, we have
`a ⊙ id == id ⊙ a == a` for any `a`. This is what makes `id` an
identity.

A semigroup with an identity is a *monoid*. That's a set with an
associative operation and an identity element. Note commutativity is not
required. If the operation is also commutative, then we have a
*commutative monoid*.

If the default constructor of our type `T` creates an identiy, then we
can reimplement `fold` like this:

``` c++
template <typename It, typename Op, 
    typename T=typename std::iterator_traits<It>::value_type>
T fold(It begin, It end, Op op)
{
    T result{ };

    for (auto it = begin; it != end; ++it) {
        result = op(result, *it);
    }

    return result;
}
```

We no longer require an initial value to be passed in. That's because,
if we really want to combine all elements with a certain value, we can
do it after calling `fold`. For example we can concatenate our strings
and then append or prepend our "!"s:

``` c++
std::vector<std::string> elems = { "foo"s, "bar"s, "baz"s };

std::string folded = fold(elems.begin(), elems.end(),
    [](const std::string& x, const std::string& y) { return x + y; });
// folded is "foobarbaz"s
```

Unlike the first implementation, this works for empty sets too:

``` c++
std::vector<std::string> elems = { };

std::string folded = fold(elems.begin(), elems.end(),
    [](const std::string& x, const std::string& y) { return x + y; });
// folded is the empty string
```

This version only works if we have an identity and if the default
constructor actually creates that identity. Note identity is a property
related to the operation, so we need to be careful. The default integer
value is `0`, which is the identity for addition, so we can very well
sum numbers using the second version of `fold`. On the other hand, we
can't use it for product:

``` c++
std::vector<int> elems = { 1, 2, 3 };

int sum = fold(elems.begin(), elems.end(),
    [](int x, int y) { return x + y; });
// sum is 6

int product = fold(elems.begin(), elems.end(),
    [](int x, int y) { return x * y; });
// product is 0
```

If we default to `0` and multiply all numbers with it, our product
becomes `0`. On the other hand, if we do have a default identity, then
`fold` and `fold_right` give us the same result even if the operation is
only associative, without necessarily being commutative. That's
because, if our initial value is an identity, it doesn't matter whether
it is a left or right argument to our operation. By definition,
`a ⊙ id == id ⊙ a == a` so for a set of values like `a`, `b`, and `c`,
we get:

``` text
((id ⊙ a) ⊙ b) ⊙ c == (a ⊙ b) ⊙ c
```

From associativity, we get:

``` text
(a ⊙ b) ⊙ c == a ⊙ (b ⊙ c)
```

We can add another identity in there, since `c == c ⊙ id`:

``` text
a ⊙ (b ⊙ c) == (a ⊙ (b ⊙ (c ⊙ id)))
```

We started with how `fold` evaluates and ended up with how `fold_right`
evaluates. That means we don't distinguish between a left to right or
right to left fold is either:

* The operation is associative and commutative (abelian semigroup).
* The operation is associative and the initial value is the identity
  (monoid with identity as initial value).

If neither of these holds, it matters which way the fold happens as we
get different results.

## Parallelized Fold

Wihout going into too many details, we can also divide & conquer a fold
operation, fold subsets in parallel, then merge the results.
Associativity allows us to do so, as we can split the set of values `a`,
`b`, `c`, and `d` into `left_half = a ⊙ b` and `right_half = c ⊙ d`,
then combine the two halves for the final result
`left_half ⊙ right_half`. This is the same thing as `(a ⊙ b) ⊙ (c ⊙ d)`.
As long as the operation is associative, this is the same as
`((a ⊙ b) ⊙ c) ⊙ d` or `a ⊙ (b ⊙ (c ⊙ d))`.

Not all operations are associative though, as we saw before. Subtraction
isn't for example. `(1 - 2) - (3 - 4)` is `0`. `((1 - 2) - 3) - 4` is
`-8`. We can only parallelize folding a semigroup, not any magma.

Here, since we are talking about dividing the input set, we can assume
we have more than zero elements in a subset (otherwise we wouldn't
divide it), so whether we have an identity or not or whether we apply it
to the left or the right can be left to the top function which combines
the partial results.

## Alternatives

We covered traditional implementations of `fold` but let's go over a
couple of alternative implementations. We could say that if the set is
empty, we don't return anything, otherwise we take the first element as
our initial value:

``` c++
template <typename It, typename Op,
    typename T=typename std::iterator_traits<It>::value_type>
std::optional<T> fold(It begin, It end, Op op)
{
    if (begin == end) return std::nullopt;

    It it = begin;
    T result = *(it++);

    for (; it != end; ++it) {
        result = op(result, *it);
    }

    return result;
}
```

As long as the operation is associative, the fold direction doesn't
matter. In case we don't have any value at all, we return `nullopt` to
signal the absence of a value.

Another option is to take a default value as an argument and return that
in case we have no elements to combine, but if we do, simply ignore that
value instead of combining it with the input elements:

``` c++
template <typename It, typename Op,
    typename T=typename std::iterator_traits<It>::value_type>
T fold(It begin, It end, T def, Op op)
{
    if (begin == end) return def;

    It it = begin;
    T result = *(it++);

    for (; it != end; ++it) {
        result = op(result, *it);
    }

    return result;
}
```

This is similar to the previous implementation, but instead of an empty
optional we return a supplied value in case we don't have anything to
fold. This would be interpreted as *fold or return def*, as opposed to
our original implementation, which was *fold with init*.

## Summary

This post started with a discussion of how types can be viewed as sets
and functions as functions over those sets. We then covered a few
abstract algebra concepts as applied to the `fold` higher order
function, looking in which situations do `fold` and `fold_right` return
the same result.

* A function `T f(T, T)` is a closed binary operation on the set `T`.
* A set with a binary operation is called a magma. We can implement a
  `fold` and a `fold_right` if we have a magma. Both functions need an
  initial value.
* If the operation is also associative, we have a semigroup. Here it
  still matters whether the initial value is applied on the left or
  the right, as the results might be different.
* If the operation is also commutative, we have an abelian semigroup.
  There is no distinction between a left-to-right and a right-to-left
  fold for an abelian semigroup.
* If the operation has an identity, then we have a monoid. If we use
  the identity as an initial value, then we again have no distinctino
  between a left-to-right and a right-to-left fold.
* We can paralllize fold to take subsets of the input set, combine
  them in parallel, then combine the results. Associativity is the
  only requirement for this, so we can parallelize folding a
  semigroup.
* We also looked at a couple of alternative implementations which only
  require a semigroup for the fold direction not to matter. The first
  one returns an empty optional if there are no values to combine; the
  second returns a value supplied as argument, without otherwise
  combining it.
