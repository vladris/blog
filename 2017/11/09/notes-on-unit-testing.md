# Notes on Unit Testing

This post covers my view on unit testing, why they are important, how to
make time for them, how much to test, and why I don't believe in TDD.
It draws from my personal experience working on multiple software
projects, small and large, both on new code and legacy code, practices I
tried to apply, what worked well and not so well. While it is more
highlevel and I do not provide code snippets, this is by no means a
purely theoretical essay.

## The "not enough time" fallacy

Engineers and engineering teams who are not "bought" on unit testing
use the excuse that there is not enough time to write unit tests. There
is always a deadline or pressure to ship and unit tests, not being
"product" code, get lower priority.

The problem is that we never get it right the first time - as we get a
better understanding of our problem space, we need to refine our
solution, which includes refactoring to better structure the code and
redesign to accommodate for new requirements. This is where unit tests
become invaluable in ensuring that such radical alterations of the
codebase can be made safely and easily.

What I noticed first hand is that a team who is not disciplined about
testing starts by churning out a significant amount of code but over
surprisingly little time development slows down to a crawl because there
is a lot of manual testing involved in validating any change, nasty bugs
come up, and since refactoring is now scary (who knows what will
break?), engineering debt keeps building up. On the other hand, teams
that author unit tests from the very beginning can maintain a steady
development pace indefinitely.

The funny thing is that engineers who worked in a code base with good
test coverage could never go back - they immediately see the benefits
and are sold on the practice - while engineers who haven't done it know
the theory, pay lip service to it, but never have time to actually
implement tests.

## Making time

There is, unfortunately, never enough time to do the right thing. My
advice is to make time: unit tests are part of feature development, so
they should be accounted for as such. Do not have separate tasks for
implementing the feature and writing the unit tests - these unit testing
tasks are prime candidates to be deprioritized and cut, after all, the
feature *works*, right? Instead consider testing as part of
implementation, create a single task, and adjust estimates accordingly.
There is always pressure to ship, the job of a good engineer is to not
cave under this pressure, set expectations, and deliver a robust
solution. As mentioned above, the ability to keep a steady development
pace makes the average cost of authoring tests over time seem like
nothing compared to the alternative - a steady drop in development
agility.

Advice to managers is to encourage a culture of quality and best
practices. Strategically, shipping next week vs the week after is not as
important as shipping in a couple of months vs shipping in a year, which
is where the brittleness of the codebase becomes a major factor. Reward
good engineering practices and you end up with well-engineered code.

That being, sometimes we *do* need to ship next week.

## MQs

In the old waterfall development days, we had several major milestones,
each spanning months of development: M1, M2 etc. As ship date came near,
pressure increased, and shortcuts were taken more often. In the end,
ship dates were met, but with a lot of compromises. What followed right
after, when the team was burned out after the final stretch, was the so
call "quality milestone" or MQ. Here, engineers were free to reduce
debt while project managers went to define the future version of the
product.

I personally love the concept of MQ. While I don't doubt the existence
of *purely agile* teams where everything is delivered after week-long
sprints with high quality, most businesses make promises to customers
and must meet deadlines. Sometimes the pressure increases enough that we
knowingly take engineering shortcuts:

> If I had a week, I'd do it the right way, but this works for now.
> I'll come back and fix it later.
>
> *- Every programmer in the world at some point*

After a hard deadline it's the perfect time to schedule a mini-MQ -
spend a week or two recovering from burnout and reducing debt.

This expands beyond unit tests to things like refactoring and
rearchitecting code, automating manual processes, writing documentation
etc.

## The Test-Driven Development fallacy

The other extreme is test-driven development. The premise of test-driven
development is that turning requirements into unit tests, then writing
code to make those tests pass is a solid approach to engineering. This
sounds great in theory but falls flat in practice.

Good software is correct, efficient, and maintainable. These qualities
come from a good understanding of the problem being solved and a
thought-through solution, not by making a set of test cases pass. An
anecdote I like to reference when discussing this is the Sudoku solver.
Ron Jeffries, one of the founders of extreme programming, wrote several
blog posts in which he attempted to implement a Sudoku solver using
test-driven development
[here](http://xprogramming.com/articles/sudokumusings),
[here](http://xprogramming.com/articles/oksudoku),
[here](http://xprogramming.com/articles/sudoku2),
[here](http://xprogramming.com/articles/sudoku4),
[here](http://xprogramming.com/articles/sudoku5), and
[here](http://xprogramming.com/articles/roroncemore/). The attempt
failed. Around the same time, Peter Norvig implemented a Sudoku solver
and wrote a [blog post](http://norvig.com/sudoku.html) with a beautiful
explanation of a thorough approach to analyzing the problem and coming
up with a good algorithm to solve it. The point here is that a set of
unit tests, no matter how comprehensive, will not design an algorithm
for you. The algorithm comes from stepping back and thinking about the
problem, which a test-centric approach actively discourages.

The one good thing that TDD encourages is writing tests which initially
fail, then providing the implementation to make them pass, which ensures
the tests themselves are correct. We can always create a test that
exercises a function and then asserts a tautology
(`Assert.IsTrue(true)`), which covers the code, makes the test pass, but
provides zero value. Having a test that fails when invoked with a stub
and passes when invoked with the real implementation avoids this issue.

## Tests are about behavior, not design

Almost forgotten across the industry nowadays is that software can be
formally proven correct. A given number of passing tests can only
guarantee that for that particular set of inputs, we get the expected
output - which, in case of test bugs, might not mean anything. The way
to be 100% confident that the code does what we think it does is to
prove this fact formally. This is not always feasible at scale, but for
critical pieces of functionality, formalism is better than test cases.

That doesn't mean tests are not needed - as soon as a line of code
changes (bug fix, optimization, etc.), the formalism must be
re-evaluated, and, outside fringe programming languages, we can't
automatically detect when a proof no longer holds. The point is that
tests are about behavior not about design - we design to solve the
problem, we test to make sure that our solution does what we expect it
to do. **Design comes first, tests come second, implementation is
third.**

Unit tests become valuable when we can make deep changes within our code
and ensure there is no observable change in output. This is invaluable
to engineering velocity.

## How much is enough?

In terms of code coverage, I believe something around 90% can easily be
achieved with a minimum of effort. Full coverage is unrealistic because
the code always has some interaction with the world - making network
calls, relying on time, random numbers, IO etc. These are all
interactions that can sporadically fail and unit tests, by definition,
must be 100% reliable. A testable design abstracts all the world
interactions under interfaces that can be mocked during testing. This
way, we end up with a thin layer that implements these interfaces and
forwards to the real OS/library functions. This thin layer should not
contain any logic beyond forwarding arguments since it is not really
testable and attempting to write unit tests against it ends up with an
ongoing cost of analyzing random test breaks due to failures in
components outside of our control. The other place where ROI is small is
testing trivial code like getters/setters. This is wasted engineering
effort and provides questionably little value. That being said, this
layer should be at most 10% of the code base, more likely somewhere in
the 1-2% range for larger projects. Everything else should be covered by
unit tests.

There is also an interesting distinction between explicit vs implicit
testing -a function can be covered explicitly, by writing unit tests
against it, or implicitly, by writing unit tests against other functions
that end up calling this function. A good rule of thumb is to test
against the interface not against the private implementation. If you
can't reach the same amount of code coverage by testing the public
interface as you can by testing the implementation details, it means you
have dead code in the implementation - code that cannot be reached from
the public interface for any possible input. This code should be removed
not tested. Unit tests have a cost themselves - if we have tests
exercising a function and, during a refactoring, we change the signature
of that function, we have to go update all these tests. If any
refactoring we make breaks unit tests and requires us to fix them,
engineering cost of maintaining test coverage is increased needlessly.

Ideally, we should break and have to update tests when we break the
interface (the unit's contract to the outside world). We should be able
to freely move the implementation guts around, as keeping tests green in
this case is the ultimate purpose of unit testing - ensuring output
through the contract doesn't break during internal changes. A couple of
gotchas here: if we feel we need to test an implementation detail
because it's scarily complex, we have a code smell - that
implementation detail should be split into multiple, less scary pieces;
if we have a lot of implementation logic underneath a thin interface, we
have another smell - the component (unit) is too clever and should be
split into multiple components, which would necessarily pull some of the
code to the interface level.

The bottom line is that we can achieve +90% test coverage without taking
dependencies on implementation details.

## Ease of testing

Unit testing must be easy.

Authoring unit tests should be cheap. Running unit tests should be fast
and 100% reliable. Unit tests should be part of the engineering
inner-loop -code/compile/unit test. Code coverage should be easy to
measure. Mocking should be easy. If any of these points fall short, test
coverage suffers. Good infrastructure makes it easy to author and
execute unit tests. This is key in encouraging a team to use good
engineering practices.

The other aspect of testing cost is design - code that is well
componentized is easily testable. Monolithic code, code that implements
lots of branching for various conditions, code that directly calls
components outside of our control (network, UI etc.), are all hard to
test. This is not an excuse to bypass testing, it's a smell of the code
itself.

## Learned hopelessness

It's easy to agree with all of the above but resign yourself to the
fact that in your organization things are different - the infrastructure
is not there, the culture is not there, there is no time. I believe that
the most successful and long-lived software projects have a codebase
ridden with compromises and outdated software practices, which is not a
symptom of any problem, it's the result of implementing a successful
business. It is our duty as software craftsman to remove the compromises
and update the outdated practices, question the status quo and strive to
make things better. Write unit tests!
