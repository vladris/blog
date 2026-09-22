# Guardrails, Guidelines, and the Git Goldmine

*"Do you still look at the code?"*

It depends. If it's a vibe-coded throwaway tool, no. But what if the code is
mission critical? A better question is *how much time do* we *spend* looking at
the code (and wrangling agents into doing *the right thing*).

Five years ago, coding productivity was mostly about writing code faster. Today
the bottleneck is guiding, evaluating, and correcting AI-generated code. How do
we help agents mutate complex code more autonomously and reduce the amount of
"hand holding" needed, especially while operating over an extremely complex
piece of software? Guardrails, repo-specific guidance, and eval-driven iteration
on these should significantly reduce human toil.

## The One-Shot Illusion

Autonomous code mutations do not equate to a "one-shot" LLM call where given a
prompt, the perfect result is produced in the first iteration. That might be the
case for demos, prototypes, one-off tools and such, but we can't realistically
expect the same when working inside a mature codebase. A mature code base is
more than code. It comes with years of accumulated constraints, assumptions,
decisions, and lessons learned. The problem is not generating code, it is
generating code that respects all of these constraints.

What we *can* do is set things up such that agents can iterate independently,
understand and follow best practices, and minimize human-in-the-loop involvement
while maintaining high-quality output.

We can achieve this with guidelines and guardrails. More importantly, we can use
AI to optimize them.

## Guardrails

The more guardrails we have in place, the more confidence we have on
AI-generated code. Guardrails are a MUST to maintain velocity and code
correctness. And by "guardrails" I don't just mean tests. There's a mix of
deterministic and non-deterministic checks we can run on any piece of code.

Deterministic tests can measure correctness (unit tests, end-to-end tests,
linters etc.), performance, etc. The more test coverage we can provide, the more
we can prevent regressions and ensure new code works as intended. The good news
is test generation can also be automated using coding agents, so there's a lot
less toil to provide good coverage.

We can also have AI agents review the code alongside humans. It's already common
practice to have agents do adversarial review of code changes. We can evolve
this to specialized agents targeting specific aspects of a proposed change: Is
the architecture sound? Are all constraints taken into account? Are standard
patterns being followed?

Coding and review agents can run in a loop and converge to a good solution
before human oversight is required.

We can't expect a coding agent, even one using the latest model, to perform a
non-trivial code mutation in one shot. Any piece of software with some history
and large user base falls into this category. It takes a lot of care, knowledge,
and expertise to refactor functionality while taking into account all the
different subsystems and orthogonal concerns, and make sure things work just as
the users want them to work. We can't expect a model to do this correctly in one
go. But we can offer it guidelines and automate iterations.

## Guidelines

We are still developing best practices on how to have coding agents work
autonomously in a code base. This is a good reference article: Harness
engineering: leveraging Codex in an agent-first world | OpenAI.

For example: co-locating documentation with code is better than having it in a
separate wiki. Agents are good at reading and following documentation, as long
as it is discoverable. Architectural decisions should also live in git, next to
the code. The more information we make available, the better an agent can
perform. Make sure any knowledge that is not explicitly captured in code is
easily discoverable.

Beyond this, we can also author reusable instructions to focus agents. This
applies both to the guardrails and to the coding agent.

On one hand, we can provide a fine-tuned set of prompts for code review of
various aspects. A subagent can do an architectural integrity review pass while
another subagent can do a coding convention pass etc. Going back to the one-shot
illusion - we can't expect a simple prompt like "review this code" will surface
all possible issues in the various dimensions we care about. But we can achieve
this with several fine-tuned prompts, each executed by a specialized subagent.

On the other hand, we can provide fine-tuned instructions to the coding agent
itself to ensure important aspects that one *should* keep in mind while making a
code change in a particular area are loaded into the context. Again, we can't
expect it to always and fully infer the different dimensions we care about, but
we can explicitly tell it what these are and what it must consider while working
on the code.

## The Git Goldmine

We usually think of git history as something we consult when debugging.

We should start thinking of it as training data.

Large repos generate historical evidence that can be used to improve both
guidance and agent behavior over time. We can treat this as a dataset in itself
and use it to improve both guidelines and agent behavior.

This is also true for pull requests: the comments, the back-and-forth, the
changes triggered by these are all insights into what should be implemented, how
it should be implemented, and what *shouldn't* be done.

These are rich sources of data that already exist for any sizable project.

## Code Mutations as a Data Science Problem

Code review agent prompts can be synthesized based on existing code review
feedback in a repo's history. PR feedback can be aggregated and condensed by
theme into the recurring concerns developers care about.

We can also build evals for code mutations: based on git history, we can look
at:

1. The state of a piece of code before a change.
2. The state of the same piece of code after a change (and subsequent bugfixes).

Assuming the final state is a useful proxy for a good outcome (in settled code),
we can run an eval by giving a coding agent the rewound git to before the change
(1), ask it to implement the feature, and grade its result against the final
state (2).

Historical changes give us examples of what good outcomes look like, allowing us
to iteratively refine guidance and prompts. With a setup like this, we can
automate a hill climb that optimizes both the reviewer and the coding agent
prompts until execution converges towards the ideal solution.

Again, the size of the code base is important – the good news is the larger the
repo, the richer the dataset. With enough code history to look at, we have
enough data to avoid overfitting to specific cases and to validate against
unseen examples.

The future of coding is not just better models. It includes the systems we build
around them: guardrails that help us verify outcomes, repo-specific guidance
that helps agents achieve better outcomes, and feedback loops for convergence.
The organizations with the richest engineering history may have the biggest
advantage here.
