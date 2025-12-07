# 10 Years

This post marks the tenth year of my technical blog. I started in 2016, ended up
writing exactly 9 posts during that year, and made it a point to consistently
stick to that number. You are reading post #90. This will be a self-indulgent
look back at the past decade.

Why 9 posts per year? While sometimes I have plenty of things to write about,
other times I get busy and don’t have the time. Maybe something like a post per
month would’ve been nicer, but harder to achieve. I’m still happy I was able to
stick to this for so long.

I started self-hosting the blog on my own engine, Tinkerer, which is now
archived [here](https://github.com/vladris/tinkerer). Tinkerer relied heavily on
[Sphinx](https://www.sphinx-doc.org/en/master/) and
[Jinja](https://jinja.palletsprojects.com/en/stable/), relying on the Sphinx
plugin system to turn it into a blog. Extensions provided post metadata,
ordering and pagination, RSS-feed generation and so on. That worked well until
it didn’t. Updates to Sphinx irrevocably broke things, plus Markdown was gaining
momentum - who even remembers the cumbersome ReStrctured Text syntax? I
implemented Baku, the spiritual successor to Tinkerer, and used PanDoc to
migrate my old posts from RST to Markdown.

Baku, which I’m using to this day, is [here](https://github.com/vladris/baku).
It completely drops the Sphinx dependency and replaces Jinja with a 100 LoC
templating engine which I’m still very proud of - my
[VerySimpleTemplate](https://github.com/vladris/baku/blob/main/baku/templating.py).
Lesson learned: the fewer dependencies the better.

Throughout the years I covered various topics I was digging into at the time.
When I started the blog in 2016, I was working on the Office C++ clients. The
very first two posts are based on a Clean Code talk I used to do (see [Clean
Code Part - 1](https://vladris.com/blog/2016/01/04/clean-code-part-1.html) and
[Clean Code Part -
2](https://vladris.com/blog/2016/01/07/clean-code-part-2.html)).

I was digging deep into modern C++ at the time. Some posts, like [Arguments and
Smart
Pointers](https://vladris.com/blog/2016/03/11/arguments-and-smart-pointers.html)
and [Notes on Types](https://vladris.com/blog/2016/10/16/notes-on-types.html)
provide best practices. Others covered some of my own R&D. A version of the
efficient stack-allocated data structure I describe in [(Ab)using
Maps](https://vladris.com/blog/2016/04/24/abusing-maps.html) made its way into
the Office core libraries and became a standard for applicable scenarios. The
[Composable
Generators](https://vladris.com/blog/2016/10/09/composable-generators.html) post
became this small library that enables generator chaining in C++:
<https://github.com/vladris/pipe>. This was back when generators were still an
experimental language feature.

Around that time, I was starting to dig into types and type systems. I wrote an
initial [Notes on
Types](https://vladris.com/blog/2016/10/16/notes-on-types.html) post, and a
while later one covering the Idris programming language - [Idris: Totality,
Dependent Types,
Proofs](https://vladris.com/blog/2017/07/20/idris-totality-dependent-types-proofs.html).
Then [Notes on OOP](https://vladris.com/blog/2018/01/27/notes-on-oop.html) and
[Clean Code: Types](https://vladris.com/blog/2018/09/09/clean-code-types.html),
all around the topic of types. This became my first published book, [Programming
with Types](https://www.manning.com/books/programming-with-types) in 2019.

Several posts from around that time were topics that I was researching and which
made their way in the book, like [Arithmetic Overflow and
Underflow](https://vladris.com/blog/2018/10/13/arithmetic-overflow-and-underflow.html)
and [Notes on Encoding
Text](https://vladris.com/blog/2018/11/18/notes-on-encoding-text.html), or
excerpts from the book once it was ready:

* [A Switchless State Machine](https://vladris.com/blog/2019/07/16/a-switchless-state-machine.html)
* [Common Algorithms](https://vladris.com/blog/2019/08/10/common-algorithms.html)
* [Higher Kinded Types: Functors](https://vladris.com/blog/2019/09/06/higher-kinded-types-functors.html)
* [Higher Kinded Types: Monads](https://vladris.com/blog/2019/09/07/higher-kinded-types-monads.html)
* [Variance](https://vladris.com/blog/2019/12/27/variance.html)

I switched jobs in 2019, going from Office to Azure and trading in C++ for data
platform architecture. My first post on the subject was [Notes on Data
Engineering](https://vladris.com/blog/2019/12/08/notes-on-data-engineering.html).
It was a new space for me, with plenty to learn, and during 2020 I wrote a bunch
about it:

* [Self-Serve Analytics](https://vladris.com/blog/2020/02/01/self-serve-analytics.html)
* [Azure Data Explorer](https://vladris.com/blog/2020/03/01/azure-data-explorer.html)
* [Machine Learning at Scale](https://vladris.com/blog/2020/04/27/machine-learning-at-scale.html)
* [Data Quality Testing Patterns](https://vladris.com/blog/2020/11/13/data-quality-testing-patterns.html)
* [Changing Data Classification Through Processing](https://vladris.com/blog/2020/11/27/changing-data-classification-through-processing.html)
* [Ingesting Data](https://vladris.com/blog/2021/03/12/ingesting-data.html)
  [Machine Learning on Azure Part
  1](https://vladris.com/blog/2021/09/10/machine-learning-on-azure-part-1.html),
  [Part
  2](https://vladris.com/blog/2021/09/17/machine-learning-on-azure-part-2.html),
  and [Part
  3](https://vladris.com/blog/2021/09/24/machine-learning-on-azure-part-3.html)

Like the previous type rabbit hole, I ended up publishing [Data Engineering on
Azure](https://www.manning.com/books/data-engineering-on-azure) in 2021.

At the same time, I switched teams again, moving from Azure back to Office to
work on [Fluid Framework](https://fluidframework.com/) and then
[Loop](https://aka.ms/loop/). I wrote the first [Mental
Poker](https://vladris.com/blog/2021/12/11/mental-poker.html) post in December
2021, while thinking through how one could implement a game over Fluid
Framework. This became an on-and-off side project which resulted into the Mental
Poker series of posts:

* [Mental Poker Part 0: An Overview](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html)
* [Mental Poker Part 1: Cryptography](https://vladris.com/blog/2023/03/14/mental-poker-part-1-cryptography.html)
* [Mental Poker Part 2: Fluid Ledger](https://vladris.com/blog/2023/06/04/mental-poker-part-2-fluid-ledger.html)
* [Mental Poker Part 3: Transport](https://vladris.com/blog/2023/11/28/mental-poker-part-3-transport.html)
* [Mental Poker Part 4: Actions and Async Queue](https://vladris.com/blog/2024/03/16/mental-poker-part-4-actions-and-async-queue.html)
* [Mental Poker Part 5: State Machine](https://vladris.com/blog/2024/03/22/mental-poker-part-5-state-machine.html)
* [Mental Poker Part 6: Shuffling Implementation](https://vladris.com/blog/2024/04/07/mental-poker-part-6-shuffling-implementation.html)
* [Mental Poker Part 7: Primitives](https://vladris.com/blog/2024/06/12/mental-poker-part-7-primitives.html)
* [Mental Poker Part 8: Rock-Paper-Scissors](https://vladris.com/blog/2024/06/24/mental-poker-part-8-rock-paper-scissors.html)
* [Mental Poker Part 9: Discard Game](https://vladris.com/blog/2024/07/18/mental-poker-part-9-discard-game.html)
* [Mental Poker Part 10: Conclusions](https://vladris.com/blog/2024/10/28/mental-poker-part-10-conclusions.html)

I wrote these over the span of two years, from early 2023 to late 2024, while
creating this open source library:
<https://github.com/vladris/mental-poker-toolkit>.

Before Mental Poker, I spent 2022 digging into computability. I always found the subject fascinating:

* [Computability Part 1: A Short History](https://vladris.com/blog/2022/02/12/computability-part-1-a-short-history.html)
* [Computability Part 2: Turing Machines](https://vladris.com/blog/2022/04/03/computability-part-2-turing-machines.html)
* [Computability Part 3: Tag Systems](https://vladris.com/blog/2022/05/20/computability-part-3-tag-systems.html)
* [Computability Part 4: Conway’s Game of Live](https://vladris.com/blog/2022/06/11/computability-part-4-conway-s-game-of-life.html)
* [Computability Part 5: Elementary Cellular Automata](https://vladris.com/blog/2022/07/06/computability-part-5-elementary-cellular-automata.html)
* [Computability Part 6: Von Neumann Architecture](https://vladris.com/blog/2022/07/31/computability-part-6-von-neumann-architecture.html)
* [Computability Part 7: Machine Implementation Details](https://vladris.com/blog/2022/09/02/computability-part-7-machine-implementation-practicalities.html)
* [Computability Part 8: Lambda Calculus](https://vladris.com/blog/2022/10/14/computability-part-8-lambda-calculus.html)
* [Computability Part 9: LISP](https://vladris.com/blog/2022/12/01/computability-part-9-lisp.html)

After wrapping up the Mental Poker toolkit, I spent 2025 implementing a text
editor and wrote several DevLog posts about it -
[Flow](https://vladris.com/blog/2025/03/08/devlog-1-flow.html),
[Formatting](https://vladris.com/blog/2025/04/04/devlog-2-formatting.html),
[Commanding](https://vladris.com/blog/2025/05/08/devlog-3-commanding.html),
[Theming and
Footguns](https://vladris.com/blog/2025/08/03/devlog-4-theming-and-footguns.html),
[Markdown and
WYSIWYG](https://vladris.com/blog/2025/09/08/devlog-5-markdown-and-wysiwyg.html).

Sometime in 2023, when GPT3 took the world by storm, I wrote another book,
[Large Language Models at Work](https://www.amazon.com/dp/B0CLSSM8RL). I didn’t
publish excerpts on my blog as I made it freely available at
<https://vladris.com/llm-book>. Fun fact, the book website is also built using
Baku, just like this blog.

The industry moved fast, and what was state of the art in 2023 became mostly
obsolete by 2025. I reflect on that in [Agent Integration
Patterns](https://vladris.com/blog/2025/09/29/agent-integration-patterns.html).

With the holiday season coming, I’m excited about [Advent of
Code](https://adventofcode.com/). I’ve been solving puzzles since its inception
and since 2023, I made it a tradition to start the year with a post covering the
past December’s Advent of Code, focusing on the problems I found most
interesting.

Not everything requires series of posts, sometimes I write a self-contained
just-for-fun thing. Like how I implemented a [digital pendulum
clock](https://vladris.com/blog/2025/06/08/digital-horology.html), a solver for
an iPhone game called [Kami 2](https://vladris.com/blog/2018/04/15/kami-2.html),
or a solver for the game [24](https://vladris.com/blog/2017/08/13/24.html). Or
that time I implemented
[Timsort](https://vladris.com/blog/2021/12/30/timsort.html) to get a better
sense of how it works.

The above is not a complete list of what I wrote here over the past 10 years,
rather a recap of some of the major themes. It’s been a fun ride so far. As for
the next 10 years - who knows? I know my next post will cover Advent of Code
2025. The sky is the limit beyond that.

I have plenty of things I still want to cover:

* More on Flow - Turns out you learn a lot when you implement a text editor.
  Also on Flow’s successor. Electron is too much of a hog so I started
  experimenting with an editor based on Tauri, geared towards long-form writing.
* More on AI - We’re still evolving the ways we integrate AI into software and
  there’s plenty of learning to cover there.
* Webdev - For the past 5 years or so, my day job had me using TypeScript,
  React, and the wonderful JS ecosystem. I have many thoughts here.
* Building product - I scratched the surface of this with [Shipping a
  Feature](https://vladris.com/blog/2021/08/12/shipping-a-feature.html). There’s
  a lot more to cover. I learned a lot shipping software to millions of users
  while working on Office.
* Other areas of interest, like programming languages, compilers and runtimes,
  pragmatic functional programming etc.

Happy holidays and see you in 2026! Thank you for reading!
