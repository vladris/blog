# Programming with Types

I'm happy to announce the early access launch of my book, [Programming
with Types](https://www.manning.com/books/programming-with-types).

This is the culmination of several years of geeking out on type systems
and software correctness. I've always liked to learn how to write
better code, but if I were to point out exactly when I started down this
particular rabbit hole, I'd say it was 2015.

I was switching teams at that point and decided to get up to speed on
modern C++. I started by watching C++ conference videos and was
mind-blown by the [C++
Seasoning](https://channel9.msdn.com/Events/GoingNative/2013/Cpp-Seasoning)
talk Sean Parent gave at Going Native a couple of years before. That
gave me a completely different perspective on generic programming.

On a parallel thread, I was playing with Haskell and learning about the
advanced features of its type system. Programming in a functional
language makes it obvious how some of the features taken from granted in
such languages get adopted by more mainstream languages as time goes by.
Closures, sum types, and monads are slowly making their way to the
mainstream.

## Bibliography

In awe of the elegance of generics and trying to learn more about them,
I picked up Stepanov's [From Mathematics to Generic
Programming](https://www.goodreads.com/book/show/23498372-from-mathematics-to-generic-programming)
and [Elements of
Programming](https://www.goodreads.com/book/show/6142482-elements-of-programming).
I realized I need a refresher on abstract algebra, so I took a detour
and went through Pinter's [A Book of Abstract
Algebra](https://www.goodreads.com/book/show/8295305-a-book-of-abstract-algebra).

Getting a sense of the mathematical underpinnings of generics made
things much clearer for me. I wanted to get a similar understanding of
the math underlying the Haskell type system, namely category theory. A
great resource on category theory is Bartosz Milewski's [Category
Theory for
Programmers](https://www.goodreads.com/book/show/33618151-category-theory-for-programmers).

Deeper still down the rabbit hole, I picked up Benjamin Pierce's famous
[Types and Programming
Languages](https://www.goodreads.com/book/show/112252.Types_and_Programming_Languages).
The book covers many aspects of type systems, from basic types and
function types, to subtyping, generics, and higher-kinded types. In
fact, this is exactly the progression my own book follows. Types and
Programming Languages is geared towards compiler writers. I wanted to
write something that can benefit any developer.

## From Theory to Practice

While learning more and more about type systems, I could tell the code I
was writing at work became better. There is a direct link between the
more theoretical realm of type system design and the day-to-day
production software. This isn't a revolutionary discovery - all fancy
type system features exist to solve some real-world problems.

I had new insights which I could use and did my best to share them. I
started this blog and posted about various practical applications. I did
hundreds of code reviews and applied what I learned there. And here is
where I found my niche.

Not every practicing programmer has the time and patience to read
highly-theoretical books, with mathematical proofs. On the other hand, I
could tell that my time wasn't wasted reading such books - they made me
a better software engineer. I figured there is room for a book which
covers type systems and the safety features they provide, with practical
applications anyone can use in their day jobs.

## Programming with Types {#programming-with-types-1}

The book starts with basic types and some of their common pitfalls:
numerical types tend to overflow or are subject to rounding errors,
strings have several encodings, and manipulating them naively causes all
sorts of issues. There are lesser known basic types, like the empty type
and the unit type, which are not as popular for some reason, even though
they have great applications: as return types for functions which never
return, or don't return anything meaningful.

After basic types, the book covers composition. Why record types are
generally better than tuples, what algebraic data types are about, and
countless applications of sum types. Functions should return either a
valid value or an error, never both. The variant design pattern,
enabling double-dispatch, looks different today than it did a few years
ago.

Function types are discussed at length, from lambdas and the functional
staples `map`, `filter`, and `reduce`, to modern features of programming
languages like `yield` and `async`/`await`. The book shows modern takes
on the strategy and decorator design patterns, implemented more
succinctly using function types.

Subtyping is another major topic, covering not only the elements of
object oriented programming and how to use them effectively, but also
variance, top types, and bottom types. For example, we can use a bottom
type to produce a value out of nowhere. Mixins are controversial, as
they are usually implemented as multiple inheritance, but I believe they
are extremely useful when designed correctly.

The next major topic is generic programming. Generic data structures are
responsible for shaping the data, while algorithms are responsible for
processing data. Iterators are a bridge between data structures and
algorithms, allowing us to mix-and-match them. As an example, we can
find an item in a tree using the same code we use to find an item in a
list.

Finally, the book covers higher-kinded types. These are higher-level
abstractions, generic types with generic arguments, which underpin
concepts like functors and monads. The joke goes that as soon as you
understand monads, you lose the ability to explain them. I'm taking it
as a challenge.

Check out my book
[here](https://www.manning.com/books/programming-with-types).
