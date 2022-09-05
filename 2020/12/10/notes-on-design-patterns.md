# Notes on Design Patterns

> *Patterns mean "I have run out of language."* --- Rich Hickey.

Many junior developers want to improve their software design skills by
studying design patterns. I was there too, of course. I believe there is
a big misconception of what design patterns *are*, and I believe we are,
indeed, over-indexing on them when we are thinking of software design.

It is very easy to over-design things, and if blindly apply design
patterns, we end up with code like the [FizzBuzz Enterprise
Edition](https://github.com/EnterpriseQualityCoding/FizzBuzzEnterpriseEdition),
which in real life translate into incomprehensible software that burns
hundreds of developer-hours for even tiny changes.

## A good design by any other name...

So what are design patterns? A common definition is:

> A software design pattern is a general, reusable solution to a
> commonly occurring problem within a given context in software design.

A lot of design patterns criticism centers around how, using some
non-object oriented language, you can express design patterns from the
Gang of Four succinctly within the language syntax, without having to
code extra scaffolding. This is why I believe there is a misconception
around what design patterns really are. My take on design patterns is
that they provide good solutions to software design problems, for
dimensions like encapsulation, decoupling etc. These patterns are
**not** their representation in any particular language.

In some instances, a language can express the same idea more succinctly.
That doesn't mean the pattern is useless. We are still modeling complex
domains with code, and we still need to account for all aspects of good
design so we don't end up with a jumbled mess. In other words, you can
write bad code in any programming language.

This is a recurring topic in my book [Programming with
Types](https://www.manning.com/books/programming-with-types), where I
show alternative implementations to the strategy pattern, the decorator
pattern, and the visitor pattern. The first two have more succinct
functional implementations, while the last one can be better
encapsulated using a discriminated union type. Regardless of how we
express the design, we are still solving the same problem. Which is why
I believe over-indexing on learning design patterns as code recipes is a
mistake.

## Smelling software rot

As a young developer wanting to learn design patterns, I stumbled upon
the [Agile Principles, Patterns, and Practices in
C#](https://www.goodreads.com/book/show/84983.Agile_Principles_Patterns_and_Practices_in_C_)
book, probably because it contained "patterns" in the title. But this
book is a real gem on good design, and I highly recommend it.

Chapter 7 talks about design smells. Design smells are a sign that the
software is rotting and might need refactoring. Here are a few examples
from the book:

* **Rigidity** - because of poorly-structured dependencies, a small
  change in one part of the code causes a cascade of subsequent
  changes in other parts of the code.
* **Viscosity** - when modifying the code, it is easier to add a hack
  rather than to implement a desing-preserving change. In other words,
  it is easier to do the wrong thing than it is to do the right thing.
* **Needles Repetition** - copy/pasted code, often with minor
  modifications, which makes minor updates very difficult to apply
  across the code base.

There are a bunch more in the book and plenty more documented online.

Developing a nose for design is, in my opinion, a lot more important
than *knowing* patterns. And while you can read about common smells,
nothing beats experience (much like reading descriptions of actual
smells vs. using your nose).

When I write code, sketching out a solution to some problem, I refactor
it several times before I submit a pull request. It is an iterative
process - I try something out, notice something off with the design,
refactor to improve and simplify.

Instead of bringing a set of prefabricated solutions and checking to see
which one fits the problem best, we can focus on refactoring smells
away. This avoids the FizzBuzz Enterprise Edition problem. The
Enterprise Edition of FizzBuzz is full of patterns! But it is needlessly
complex. Accidental complexity is one of the worst smells. I mentioned
accidental complexity in the [Time and
Complexity](https://vladris.com/blog/2020/01/19/time-and-complexity.html)
post and I will probably write more about it since I find this a
fascinating topic.

Smells tell us how a design is bad, but what makes a good design?

## SOLID principles and beyond

There is a small set of design principles, known as the SOLID design
principles which make for good code:

* **The single-responsibility principle** - a class (or program unit)
  should be responsible for one thing, and have to change only when
  the requirements of that thing change.
* **The open/closed principle** - code should be open for extension,
  closed for modification.
* **The Liskov substitution principle** - replacing an instance of
  some type with an instance of a subtype should maintain program
  correctness.
* **The interface segregation principle** - single-responsibility
  principle applied to interfaces.
* **The dependency-inversion principle** - code should depend on
  abstractions, not concrete implementations.

While a lot of the literature is centered around object-oriented
programming, these principles transcend OOP. For example, an interface
can be an `interface` definition, a function signature, a set of APIs
exposed by a module etc. Similarly, subtyping does not imply
inheritance - check out my
[Variance](https://vladris.com/blog/2019/12/27/variance.html) post which
covers this in more depth.

These SOLID principles are the subject of chapter 8 through 12 of the
*Agile Principles, Patterns, and Practices in C# book*.

Knowing these principles and having a nose for design smells, you can
derive any design pattern from first principles. In some cases it's
easier if you are aware of the pattern - you don't have to spend time
solving an already-solved problem. But it doesn't work the other way
around: You can't start with a set of patterns without understanding
the underlying principles and without being able to tell when a design
smells.

Beyond these principles, exploring well-crafted software helps us
develop a sense of good design. In the real-world, most codebases have
good and bad parts. I won't go into the details of why in this article,
but keep this in mind when working on your project. Which parts are
designed well? Why? Which parts could benefit from a refactoring? Why?

Learning design patterns is secondary to understanding good and bad
design.
