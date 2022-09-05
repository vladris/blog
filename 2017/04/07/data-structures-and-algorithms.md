# Data Structures and Algorithms

"Data Structures and Algorithms" is one of the basic CS courses. Data
structures and algorithms are also the building blocks of software. In
this post I will give a quick overview of data structures, algorithms,
and cover *iterators*, which bridge the two together.

## Data Structures

As the name implies, data structures provide a way of structuring data,
in other words, they maintain some set of relationships between the
contained elements. Data structures are built around expected
access/update patterns and encapsulate the inherent tradeoffs. For
example, a queue models FIFO access (so accessing the last inserted
element requires dequeuing all elements which is `O(n)` for `n`
elements) while a stack models the opposite LIFO access (which can
access the last inserted element in `O(1)` but conversely makes
accessing the first element `O(n)` for `n` elements). A deque allows
elements to be inserted and removed from both front and back, but not
from the middle. On the other hand, inserting an element in a forward
list (where each node starting form head has a pointer to its successor)
can be done anywhere, but requires a traversal of the data structure up
to the insertion point (`O(n)`).

More complex data structures exist which model more complex
relationships between elements, for example graphs and trees.

In practice, while there are always complex situations which require the
use or development of exotic data structure, I consider those to be
exceptions - a few basic data structures are enough to solve most
problems. In fact, in most cases, simple linear data structures like
lists are sufficient.

It's worth noting that the relationships and access patterns modeled by
a data structure do not have anything to do with the type of the
contained data. A queue of integers or a queue of strings work in
exactly the same way. Generics provide a great mechanism to separate the
organizing structure from the data itself. Thus the C++ `std::vector<T>`
can provide a generic implementation of a heap array for any type `T`,
the same way a C# `List<T>` does. These generic data structure model how
the contained elements are laid out, but work with any provided type.

## Algorithms

The dictionary definition of an algorithm is:

> noun: a process or set of rules to be followed in calculations or
> other problem-solving operations, especially by a computer.

There is a set of basic functions we can put our data through: search,
partition, rotate, sort, map, reduce, filter and so on. These functions
can be implemented in several ways, depending on the characteristics of
the input. For example, we can search for an element in `O(log n)` time
if our input is sorted and we can access it from the "middle" at no
extra cost. On the other hand, given unsorted data or a data structure
like a forward list which we can only access through its head, search
becomes an `O(n)` operation. The implementations of these functions are
what we call algorithms. In the examples above the algorithms are
*binary search* and *linear search*.

The same observation as with data structures applies: while there are
complex problems which require the development of brand new algorithms,
in practice, the vast majority of processing that we want to perform on
our data can be expressed either as a simple algorithm or a composition
of simple algorithms.

It is also interesting to note that the algorithms themselves are not
tied to a particular data type either, rather they only require certain
characteristics of their input. So we can perform a search as long as
there is some equivalence relation defined for the input data.
Similarly, we can perform a sort as long as there is a total order
relation defined on the input type. It doesn't really matter whether we
search for numbers or strings, the steps we take are the same.

Generics help here too, since they allow us to conceptually separate the
implementation of the algorithm (the steps) from the data we are
operating on. So the C++ `std::partition` algorithm can partition any
input -- given by a pair of forward iterators using any given predicate.
Similarly, the C# LINQ `Select` (known in most other languages as a
`map` operation), transforms all input values into output values given a
mapping function from the input type to the output type. We don't need
to implement a partition for ints, one for strings, and one for dates,
we need a generic partition which implements the steps of the algorithm
and works with any given data type.

## Iterators and Ranges

Iterators act as the bridge between data structures and algorithms.
Iterators traverse a given data structure in a linear fashion, such that
an algorithm can process the input in a consistent manner, regardless of
the actual layout of the data. Note the data structure itself does not
need to be a linear one: a binary tree can be traversed with a preorder
iterator, or an inorder iterator, or a postorder iterator.

Algorithms work on ranges of data, which can be defined as a pair of
iterators (beginning and end) or an iterator and the number of available
elements (beginning and length). I will cover some of the C++ iterator
concepts since they are the most fleshed out. Other languages usually
rely on a subset of these.

**Input iterators** can only be advanced and are one-pass only. These
map to input streams, for example unbuffered keyboard input where data
can be read once, but a subsequent read would yield different data.

**Forward iterators** extend input iterators to multiple passes. For
example, a forward iterator models traversal of a forward list. We can
always re-start traversal from any saved position, but we cannot move
back (since nodes only have links to successors, not predecessors).

**Bidirectional iterators** extend forward iterators to bidirectional
access. For example, a bidirectional iterator models traversal of a
doubly linked list. Here, we can move from one node in either direction
-- to its predecessor or to its successor.

**Random access iterators** extend bidirectional iterators to random
access, meaning any element can be accessed in constant time. For
example, a random access iterator models traversal of an array. Here, we
can access any element at the same cost, since we don't need to perform
any traversal, simply index into the array.

Depending on the implementation of a given algorithm, different iterator
types might be required. The same function can sometimes be implemented
with several algorithms, having a more efficient version work with more
capable iterators and an alternative algorithm for less capable
iterators. For example we can implement an `O(n log n)` quicksort with a
random access iterator but we can also implement an `O(n^2)` bubblesort
that works with forward iterators.

`IEnumerator<T>` in C# models a forward iterator. The (simplified)
interface is:

``` C#
interface IEnumerator<T>
{
    T Current { get; }
    bool MoveNext();
    void Reset();
}
```

This allows us to advance the iterator and to reset it to the initial
position and re-start traversal, which is exactly what a forward
iterator does.

Lazy evaluation in some functional languages and generators (functions
that yield results) model input iterators which can be advanced in a
single pass.

While most relevant operations can be implemented with input iterators,
the resulting algorithms are not very efficient. For example, with a
bidirectional iterator, `reverse` can be implemented in `O(1)` space by
starting from both ends and swapping elements. On the other hand, given
an input iterator, `reverse` requires `O(n)` space as elements need to
be pushed onto a stack and popped in reverse order.

## Summary

* Data structures model the relationship between elements and
  encapsulate access patterns. Generic data structures provide a good
  abstraction decoupling the structure of the data from the actual
  contianed data.
* Algorithms implement operations over data. Algorithms are grouped
  together based on the function or transformation they implement.
  Generic algorithms abstract the operational steps from the input the
  algorithms operate on.
* Iterators provide a bridge between data structures and algorithms.
  The more capabilities an iterator has (ie. the more restrictions we
  impose on the input), the more efficient the algorithm can be.
  Similarly, most operations can be implemented in less efficent
  manners (time and space-wise) on iterators with fewer capabilities
  (ie. fewer restrictions imposed on the input).

Recommended reading:

* [From Mathematics to Generic
  Programming](http://www.goodreads.com/book/show/23498372-from-mathematics-to-generic-programming)
  by Alexander A. Stepanov and Daniel E. Rose.
* [Elements o
  Programming](https://www.goodreads.com/book/show/6142482-elements-of-programming)
  by Alexander A. Stepanov and Paul McJones.
