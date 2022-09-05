# Recommendations

I sometimes get asked for learning resources and areas aspiring software
engineers should focus on. This post will cover some of my
recommendations. This is purely for software development, design, and
engineering. I will share a complementary list of resources on soft
skills, systems, leadership, and working within organizations in a
future post.

## Fundamentals

**Data structures and algorithms** -- This is CS 101. Understand lists,
stacks, queues, heaps, trees, graphs, and algorithms to sort, select,
traverse etc. Understand big-O notation and complexity -- this is
important in practice, when implementing solutions that deal with
real-world data. This is foundational to the field of computer science,
so unless you are working on some cutting-edge stuff, there's probably
a well-known solution to your problem. You don't need to know all
implementations by heart but know what applies and where to look it up
when needed, be it an A\* search or B-tree.

**Understand your compute target** -- this can be a physical machine, an
operating system, a virtual machine like the JVM or .NET CLR, the
browser, or the cloud. Either way you need to know what resources are
available, how they are allocated, what are the performance
characteristics and so on. Without understanding your compute target you
won't be able to leverage it to its full capabilities and run the risk
of misusing it.

**Concurrent programming** -- today, I believe this is unescapable.
Concurrency is everywhere, regardless of whether you are building
services that talk to each other, a multi-threaded native application,
or a Node.JS, event loop-based application. Having a good mental model
of how concurrency works, understanding deadlocks, livelocks,
synchronization mechanisms, and consistency models is foundational.

## Programming languages

Programming languages are not just how we talk to computers, they are
tools for thought. They enable us to express solutions to problems. We
model our solutions using the languages, and different languages are
best suited to different problems. I'm a firm believer in
multi-paradigm and using the best tool for a job. Saying that
*everything is an object*, or *everything is a function* is an
oversimplification. Many modern programming languages support multiple
paradigms, so we can write object-oriented code when appropriate, and
functional code when appropriate. With that said, I suggest learning:

**A system programming language** like C++, Rust or Swift if you need to
write native, performant code. These languages are close to the machine
and will help you understand OS resource management, how code gets
executed, and what impacts performance.

**A "higher-level" language** for writing tools and services.
Something like C#, Java or Go. These are all garbage collected and trade
off some performance for productivity.

**A dynamic language** for quick prototyping and experimentation. Python
and Ruby come to mind. While static typing is much safer for production
code, I do love *thinking* in Python.

**A purely functional language** like Haskell or Idris. This is again to
understand different ways of thinking about problems. Even if you don't
get to write production code in a purely functional language, you will
learn alternative approaches to designing your code which you will be
able to apply even when using other languages.

**A language with strong support for generics**, like C++, Rust, or
TypeScript. Generics are a powerful way to reuse and combine code and
understanding them will make you a better programmer.

From my personal experience, within the same paradigm, languages are
more similar than different. In other words, once you know two, it is
significantly easier to understand the third. There is value in learning
new programming languages to see how they differ from existing ones and
what they bring to the table.

## Practice

Write code. Learning needs to be a mix of theory and practice. Here are
some practice ideas:

[Code katas](https://en.wikipedia.org/wiki/Kata_(programming)) -- These
could be some good first projects when picking up a new language. For
example: <http://codekata.com/>,
<https://github.com/gamontal/awesome-katas>.

**Programming puzzles** -- These are good ways to practice problem
solving, data structures, and algorithms. I really enjoy [Advent of
Code](https://adventofcode.com/), which has been running every December
since 2015. [Facebook Hacker
Cup](https://www.facebook.com/codingcompetitions/hacker-cup/) also has
some great puzzles.

**Contribute to an open-source project**, many have supportive
communities and paths to get you started.

**Work on your own project**, be it a website or a game or something
else you are passionate about. Scope it to fit the amount of time you
must dedicate and use the technologies you want to learn.

Of course, you will do most of the **learning on the job**. Work on
projects that interest you, projects you can learn from, and projects
where you can apply what you learned.

## Books

This is a list of books on software design and craftsmanship that I
highly recommend:

[Code Complete](https://www.goodreads.com/book/show/4845.Code_Complete)
-I love this one. This is The Big Book of Software Engineering, and it
covers the fundamentals, like writing proper functions, using good
naming in the code, testing, debugging etc.

[The Pragmatic
Programmer](https://www.goodreads.com/book/show/4099.The_Pragmatic_Programmer) -
Great book which gives general advice on what it takes to become a good
programmer. From basic tips like know thy text editor and use source
control to implementation and design advice.

[Agile Principles, Patterns and Practices in
C#](https://www.goodreads.com/book/show/84983.Agile_Principles_Patterns_and_Practices_in_C_) -
On writing software in an agile world. Principles to keep in mind (like
the Open/Close Principle, the Single Responsibility Principle etc.),
patterns to enable them, and agile practices. Also covers TDD, extreme
programming, and all the other agile methodologies.

[Design
Patterns](https://www.goodreads.com/book/show/85009.Design_Patterns) -
The classic book on design patterns. It is a bit cumbersome but
definitely worth reading. Patterns are basically well-known solutions to
recurring design problems. Do read [my previous
post](https://vladris.com/blog/2020/12/10/notes-on-design-patterns.html)
and don't over-index on patterns.

[Refactoring](https://www.goodreads.com/book/show/44936.Refactoring) -
The classical book on refactoring code - re-structuring implementation
without changing functionality. Talks about code smells (pieces of code
that feel wrong) and how to rearchitect such code to make it right.

[Emergent
Design](https://www.goodreads.com/book/show/3139913-emergent-design) -
This book talks about how design emerges as code evolves. By respecting
a few principles and knowing about design patterns, you don't have to
over-design and future-proof, rather keep refactoring and extending as
new requirements come in.

There are a few other books which I really enjoyed, though they are a
bit different than the above. I recommend these more for the aesthetics
and insights (more on that below): [Programming
Pearls](https://www.goodreads.com/book/show/52084.Programming_Pearls),
[From Mathematics to Generic
Programming](https://www.goodreads.com/book/show/23498372-from-mathematics-to-generic-programming),
[Beautiful
Code](https://www.goodreads.com/book/show/405790.Beautiful_Code).

## Other considerations

**Develop a sense of aesthetics** - This comes with practice. Know what
good code looks like, what makes it beautiful. Don't just get code
working, try to make it beautiful.

**Understand your problem space** -- Whatever you are working on,
knowing the business domain will help inform your software design.
Understand why you are doing what you are doing. Is there a better way
to solve the same business problem? Do you know what will likely come up
in 6 months from now or a year?

**Security** -- Software security is critical in today's connected
world. You should understand security best practices, which hashing
algorithms to use, how to properly store secrets and passwords, how
trust gets established, attack vectors, how to create a [threat
model](https://en.wikipedia.org/wiki/Threat_model) and so on.

**AI** - AI is permeating more areas of software. It is also being
commoditized through libraries like scikit-learn and services like Azure
Cognitive Services. While I won't quite yet put it under
*fundamentals*, I believe using AI will soon be a must-know, much like
concurrent programming. Having a good understanding of the types of
problems AI can help with, when and how to apply it is very valuable.

**Keep learning** -- The way we build software keeps evolving. Try to
keep up to date with recent developments and trends. This is one of the
reasons software engineering is such an exciting field: there's always
something new, there's always more to learn.
