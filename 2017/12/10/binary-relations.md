# Binary Relations

## Definitions

Given a set of objects `A`, a binary relation `R` on the set is defined
as a subset of `A x A`. The *characteristic function* `r` for `R` is the
function `r : A x A -> bool` such that `r(x, y)` is `true` if
`(x, y) in R`, and `false` if `(x, y) not in R`. For a more natural
notation, we can use `x ~ y` to denote `r(x, y)`.

More generally, a binary relation can be defined on a pair of sets
`A x B` but to keep things simple, we'll only cover binary relations
over a single set.

Binary relations may have several properties. A few interesting ones
are:

* A binary relation is *reflexive* if for any `x` in `A`, `x ~ x`.
* A binary relation is *strict* or *irreflexive* if there is no `x` in
  `A` for which `x ~ x`.
* A binary relation is *symmetric* if for any `x, y` in `A`, `x ~ y`
  implies `y ~ x`.
* A binary relation is *antisymmetric* if for any `x, y` in `A`,
  `x ~ y` and `y ~ x` implies `x = y`.
* A binary relation is *transitive* if for any `x, y, z` in `A`, if
  `x ~ y` and `y ~ z`, then `x ~ z`.
* A binary relation is *total* if for any `x, y` in `A`, either
  `x ~ y`, `y ~ x`, or both (in other words, for any `x, y`, `~`
  imposes some relation between them).

### Examples

The relation *is in the subtree rooted at* is a reflexive relation where
`A` is the set of nodes of a tree. For any pair of nodes `x` and `y`, we
can establish whether `x` is in the subtree rooted at `y` or not, and
for any `x`, `x ~ x` is `true`.

The relation *is parent of* in a tree is a strict relation: for any `x`
in the set of tree nodes `A`, `x` cannot be a parent of itself.

The relation *edge between* over the vertices of a non-directed graph is
a symmetric relation: for any `x` and `y` vertices of the graph, if
there is an edge from `x` to `y`, the same edge exists from `y` to `x`,
in other words, if `x ~ y` then `y ~ x`.

The *is in the subtree rooted at* relation above is also antisymmetric:
if for a pair of nodes we can say `x` is in the subtree rooted at `y`
and also `y` is in the subtree rooted at `x`, it's obvious that both
`x` and `y` are, in fact, the root of the subtree, thus `x ~ y`.

The relation *is reachable from* over the vertices of a directed graph
is a transitive relation: if `x` is reachable from `y` and `y` is
reachable from `z`, then `x` is reachable from `z`.

All of the above examples are of total relations. An example of a
non-total relation is *is ancestor of* in a tree. `x` can be an ancestor
of `y`, in which case `x ~ y`, or `y` can be an ancestor of `x`, in
which case `y ~ x`, but it could also be that `x` and `y` are in
different subtrees, so neither `x ~ y` nor `y ~ x` holds.

## Preorder

A *preorder* is a relation which is reflexive and transitive.

A preorder which is also symmetric is an *equivalence*. A preorder which
is antisymmetric is a *partial order*. More on those below.

An example of preorder is the *is reachable from* relation over a
directed graph in the example above. This relation is obviously
reflexive and transitive, but it is neither symmetric nor antisymmetric.
If `x` is reachable from `y`, it doesn't mean that `y` is reachable
from `x`, so symmetry is not guaranteed. Similarly, if `x` is reachable
from `y` and `y` is reachable from `x`, it does not mean that `y` equals
`x`.

## Equivalence and Equality

An *equivalence* relation `~` is a binary relation that is reflexive,
symmetric, and transitive. In other words, it is a preorder which also
has the symmetric property.

Such a relation partitions the set over which it is defined into
*equivalence classes* - groups of objects that are equivalent based on
the relation.

An example of equivalence is *same month* over a set of dates. This
relation is reflexive, since a date `d` has the same month as itself
(`d ~ d`); is symmetric, since if `d1` has the same month as `d2`, then
`d2` has the same month as `d1` (`d1 ~ d2 => d2 ~ d1`); and transitive,
since if `d1 ~ d2 and d2 ~ d3 => d1 ~ d3`.

This relation partitions our set of dates in the equivalence classes
corresponding to *dates in January*, *dates in February*, and so on.
Note that the dates for which the relation holds are equivalent, but not
necessarily equal.

An *equality* relation is an equivalence relation which partitions the
set `A` consisting of `n` objects into exactly `n` equivalence classes.
In other words, for any `x` in `A`, only `x ~ x` is `true`.

## Partial Order and Total Order

A *partial order* relation `<=` is a binary relation that is reflexive,
antisymmetric, and transitive. In other words, it is a preorder which
also has the antisymmetric property.

An example of a partial order is the *is subset of* relation. It is
reflexive (`A` is a subset of `A`), antisymmetric (if `A` is a subset of
`B` and `B` is a subset of `A`, then `A = B`), and transitive (if `A` is
a subset of `B` and `B` is a subset of `C`, then `A` is a subset of
`C`).

A *total order* relation is a partial order that is also total. The
above example relation *is subset of* is not total - there could be a
pair of sets `A` and `B` such that neither is the subset of the other.

An example of a total order relation is *less than or equal to* for
integers.

## Weak Order and Strict Weak Order

A *weak order* relation `~` is a binary relation that is transitive and
total. This implies reflexivity (for any `x` and `y`, either `x ~ y`,
`y ~ x`, or both, so for `x` and `x` we have `x ~ x`). In other words,
it is a preorder which is also total.

An example of a weak order is *less than or equal absolute value* for
complex numbers. For any two complex numbers `c1` and `c2`, either
`c1 ~ c2` ( `|c1| <= |c2|`), `c2 ~ c1` (`|c2| <= |c1|`), or both, so `~`
is total. We also have `c1 ~ c2` and `c2 ~ c3` implies `c1 ~ c3`
(`|c1| <= |c2|` and `|c2| <= |c3|` implies `|c1| <= |c3|`). Unlike a
total order though, the relation is not antisymmetric. We can have
`c1 ~ c2`, `c2 ~ c1`, with `c1` and `c2` distinct complex numbers (any
two numbers with the same absolute value but different components).

A *strict weak order* relation `<` is a binary relation that is
transitive and strict (irreflexive).

An example of strict weak order is *less than* for integers.

## Applications

Most programming languages provide a way to customize equality,
inequality, and comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`).
There is an interesting point to be made about what equality *means* in
this context. For some types, this can simply mean comparing the bits
and if they are the same, the objects are equal. But we also have
*logical equality* - two objects can have different bitwise values but
still be considered equal. Even more so for comparing objects -
comparing bit representations usually does not translate to a meaningful
comparison of objects.

Note though that any other function `bool r(const T& m1, const T& m2)`
or member function `bool r(const T& other)` of `T` denotes a binary
relation on `T`.

Different algorithms require different types of relations to exist
between objects.

For example, we need at least a partial order relation to perform a
topological sort. That is, in an directed acyclic graph, we can sort the
vertices such that for every edge from `a` to `b`, `a` precedes `b` in
the order. This can be used, for example, on the dependency graph in a
makefile to determine how to sequence work.

Having an equivalence relation (eg. `==`), we can implement a linear
search algorithm to traverse a data structure and find an object
equivalent to a given object. The C++ standard library algorithm `find`
is an example of such an algorithm.

Having a total order relation (eg. `<=`) or a strict weak order (eg.
`<`), allows us to implement binary search over an ordered set of
objects. A total order or strict weak order relation also enables
comparison sort algorithms.

Similarly, we need a total order or strict weak order to be able to
determine a minimum or a maximum element from a set of objects
(`min_element` and `max_element` algorithms in C++).

## Summary

* A binary relation `R` on a set `A` is a subset of `A x A`, denoted
  by a characterisitc function `r : A x A -> bool`.
* A binary relation on a type `T` is denoted by either a free function
  of the the form `bool r(const T&, const T&)` or a member function
  `bool r(const T&)`.
* A binary relations may have several properties: it can be reflexive
  or strict, symmetric or antisymmetric, transitive, total etc.
* Depending on the properties it has, a relation can be, for example:
  * A preorder (reflexive and transitive).
  * An equivalence (reflexive, symmetric, and transitive).
  * A partial order (reflexive, antisymmetric, and transitive).
  * A weak order (reflexive, transitive, and total).
  * A strict weak order (irreflexive, transitive, and total).
* Certain algorithms require the types they operate on to have
  relations with certain properties.
