# Common Algorithms

This blog post is an excerpt from my book, [Programming with
Types](https://www.manning.com/books/programming-with-types). The code
samples are in TypeScript. If you enjoy the article, you can use the
discount code **vlri40** for a 40% discount on the book.

## A Few Common Algorithms

There are many algorithms commonly used to process a sequence of data.
Let's list a few of them. We will not look at the implementation, just
describe what arguments besides the iterable they expect and how they
process the data. We'll also mention some synonyms under which the
algorithm might appear.

* `map()` takes a sequence of `T` values, a function `(value: T) => U`
  and returns a sequence of `U` values applying the function to all
  the elements in the sequence. It is also known as `fmap()`,
  `select()`.
* `filter()` takes a sequence of `T` values, a predicate
  `(value: T) => boolean` and returns a sequence of `T` values
  containing all the items for which the predicate returns true. It is
  also known as `where()`.
* `reduce()` takes a sequence of `T` values, an initial value of type
  `T`, and an operation which combines two `T` values into one
  `(x: T, y: T) => T`. It returns a single value `T` after combining
  all the elements in the sequence using the operation. It is also
  known as `fold()`, `collect()`, `accumulate()`, `aggregate()`.
* `any()` takes a sequence of `T` values and a predicate
  `(value: T) => boolean`. It returns true if any one of the elements
  of the sequence satisfies the predicate.
* `all()` takes a sequence of `T` values and a predicate
  `(value: T) => boolean`. It returns true if all of the elements of
  the sequence satisfy the predicate.
* `none()` takes a sequence of `T` values and a predicate
  `(value: T) => boolean`. It returns true if none of the elements of
  the sequence satisfy the predicate.
* `take()` takes a sequence of `T` values and a number `n`. It returns
  a sequence consisting of the first `n` elements of the original
  sequence. It is also known as `limit()`.
* `drop()` takes a sequence of `T` values and a number `n`. It returns
  a sequence consisting of all the elements of the original sequence
  except the first `n`. The first `n` elements are dropped. It is also
  known as `skip()`.
* `zip()` takes a sequence of `T` values and a sequence of `U` values.
  It returns a sequence containing pairs of `T` and `U` values,
  effectively "zipping" together the two sequences.

There are many more algorithms for sorting, reversing, splitting and
concatenating sequences. The good news is that, because these algorithms
are so useful and generally applicable, we don't need to implement
them. Most languages have libraries which provide these algorithms and
more. For JavaScript, there is the `underscore.js` package and the
`lodash` package, both providing a plethora of such algorithms (at the
time of writing, these libraries don't support iterators, only the
JavaScript built-in array and object types). In Java, they are found in
the `java.util.stream package`. In C# they are in the `System.Linq`
namespace. In C++ they are found in the `<algorithm>` standard library
header.

## Algorithms Instead of Loops

While you might be surprised, a good rule of thumb is to check, whenever
you find yourself writing a loop, whether there is a library algorithm
or a pipeline that can do the job. Usually we write loops to process a
sequence - exactly what the algorithms we talked about do.

The reason to prefer library algorithms to custom code in loops is that
there is less opportunity for mistakes: library algorithms are tried and
tested, implemented efficiently, and the code we end up with is easier
to understand as the operations are spelled out.

## Implementing a Fluent Pipeline

Most libraries also provide a fluent API to chain algorithms together
into a pipeline. Fluent APIs are APIs based on method chaining, making
the code much easier to read. To see the difference between a fluent and
a non-fluent API, let's take a look at a simple filter/reduce pipeline.

Let's start with a simple implementation of the two algorithms. To
implement `filter()` we can use a generator. We take an `Itreable<T>` as
the input sequence and a predicate from `T` to `boolean`, and return
another sequence as an `IterableIterator<T>`. `ItreableIterator` is the
return type of all generators in TypeScript. The function will simply
traverse the sequence and for each element, if the predicate returns
true, yield the element to the caller:

``` ts
function *filter<T>(
    items: Iterable<T>,
    pred: (x: T) => boolean)
    :IterableIterator<T> {
    for (const item of items) {
        if (pred(item)) {
            yield item;
        }
    }
}
```

`reduce()` takes an `Iterable<T>` as the input sequence and an initial
value of type `T`. It also takes a function `(T, T) => T` which combines
(reduces) two values of type `T` into one. This function iterates over
the sequence and reduces all the elements to a single value, which it
returns:

``` ts
function reduce<T>(
    items: Iterable<T>,
    init: T,
    op: (x: T, y: T) => T)
    : T {    
    let result: T = init;

    for (const item of items) {
        result = op(result, item);    
    }

    return result;
}
```

Now let's look at how we could combine these algorithms into a pipeline
which sums up all even values of an array. We will pass the array to
`filter()` first, with a predicate which returns true for even numbers.
Next, we will reduce the resulting sequence using an initial value of 0
and the function `(x, y) => x + y`:

``` ts
const sequence: number[] = [1, 2, 3, 4, 5, 6];

const result: number = 
    reduce(
        filter(
            sequence,
            (value) => value % 2 == 0),
        0,
        (x, y) => x + y);

console.log(result);
```

Even though we apply `filter()` first, then pass the result to
`reduce()`, if we read the code from left to right, we see `reduce()`
before `filter()`. It's also a bit hard to make sense of which
arguments go with which function in the pipeline. Fluent APIs make the
code much easier to read. Currently, all our algorithms take an iterable
as the first argument and return an iterable. We can use object-oriented
programming to improve our API. We can put all our algorithms into a
class which wraps an iterable. Then we can call any of them without
explicitly providing an iterable as the first argument - the iterable
is a member of the class. Let's do this for `map()`, `filter()`, and
`reduce()`, by grouping them into a new `FluentIterable<T>` class
wrapping an iterable:

``` ts
class FluentIterable<T> {
    iter: Iterable<T>;

    constructor(iter: Iterable<T>) {
        this.iter = iter;
    }

    *map<U>(func: (item: T) => U)
        : IterableIterator<U> {
        for (const value of this.iter) {
            yield func(value);
        }
    }

    *filter(pred: (item: T) => boolean)
        : IterableIterator<T> {
        for (const value of this.iter) {
            if (pred(value)) {
                yield value;
            }
        }
    }

    reduce(init: T, op: (x: T, y: T) => T)
        : T {
        let result: T = init;

        for (const value of this.iter) {
            result = op(result, value);
        }

        return result;
    }
}
```

We can create a `FluentIterable<T>` out of an `Iterable<T>`, so we can
rewrite our filter/reduce pipeline into a more fluent form. We create a
`FluentIterable<T>`, call `filter()` on it, then we create a new
`FluentIterable<T>` out of its result, and call `reduce()` on it:

``` ts
const sequence: number[] = [1, 2, 3, 4, 5, 6];

const result: number =
    new FluentIterable(
        new FluentIterable(
            sequence  
        ).filter((value) => value % 2 == 0)    
    ).reduce(0, (x, y) => x + y);    

console.log(result);
```

Now `filter()` appears before `reduce()`, and it's very clear which
arguments go to which function. The only problem is we need to create a
new `FluentIterable<T>` after each function call. We can improve our API
by having our `map()` and `filter()` functions return a
`FluentIterable<T>` instead of the default `IterableIterator<T>`. Note
we don't need to change `reduce()`, because `reduce()` returns a single
value of type `T`, not an iterable.

Since we're using generators, we can't simply change the return type.
Generators exist to provide convenient syntax for functions, but they
always return an `IterableIterator<T>`. What we can do instead is to
move the implementations to a couple of private methods, `mapImpl()` and
`filterImpl()`, and handle the conversion from `IterableIterator<T>` to
`FluentIterable<T>` in the public `map()` and `reduce()` methods:

``` ts
class FluentIterable<T> {
    iter: Iterable<T>;

    constructor(iter: Iterable<T>) {
        this.iter = iter;
    }

    map<U>(func: (item: T) => U)
        : FluentIterable<U> {
        return new FluentIterable(this.mapImpl(func));    
    }

    private *mapImpl<U>(func: (item: T) => U)
        : IterableIterator<U> {
        for (const value of this.iter) {    
            yield func(value);
        }
    }

    filter<U>(pred: (item: T) => boolean)
        : FluentIterable<T> {
        return new FluentIterable(this.filterImpl(pred));    
    }

    private *filterImpl(pred: (item: T) => boolean)
        : IterableIterator<T> {
        for (const value of this.iter) {    
            if (pred(value)) {
                yield value;
            }
        }
    }

    reduce(init: T, op: (x: T, y: T) => T)
        : T {    
        let result: T = init;

        for (const value of this.iter) {
            result = op(result, value);
        }

        return result;
    }
}
```

With this updated implementation, we can more easily chain the
algorithms, as each returns a `FluentIterable`, which contains all the
algorithms as methods:

``` ts
const sequence: number[] = [1, 2, 3, 4, 5, 6];

const result: number =
    new FluentIterable(sequence)
        .filter((value) => value % 2 == 0)    
        .reduce(0, (x, y) => x + y);    

console.log(result);
```

Now, in true fluent fashion, the code reads easily from left to right
and we can chain any number of algorithms that make up our pipeline with
a very natural syntax. Most algorithm libraries take a similar approach,
making it as easy as possible to chain multiple algorithms together.

Depending on the programming language, one downside of a fluent API
approach is that our `FluentIterable` ends up containing all the
algorithms, so it is difficult to extend - if it is part of a library,
calling code can't easily add a new algorithm without modifying the
class. C# provides extension methods, which enable us to add methods to
a class or interface without modifying its code. Not all languages have
such features though. That being said, in most situations you should be
using an existing algorithm library, not implementing a new one from
scratch.
