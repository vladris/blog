# Agent Integration Patterns

A couple of years ago, when LLMs started taking off, I wrote a whole book about
how to integrate models into larger applications. I knew going in that the book
will get outdated very fast due to the speed at which the industry is moving. In
fact, a few months after I finished it, the model I used throughout the book for
code examples was deprecated by OpenAI. This broke all code samples. The API
itself changed significantly since then. I guess I could use a model to upgrade
them, but the way we use models today is already very different - simply getting
the code working again is not enough.

## Agents and Patterns

I won't attempt another applied AI book, not at this time, but I am writing this
blog post to reflect on some of the advancements and modern ways of leveraging
models. Beyond just chat, agents are becoming more and more capable and thus
useful. I want to focus on that.

I also believe, at some point, we will have the equivalent of the famous Design
Patterns, but focused on creating and integrating agents. We're still early, the
technology is going to advance by a few more leaps and bounds before we can pin
some things down. Or we go singularity, in which case who cares, AI will figure
everything out for us.

With the caveat that things might still change dramatically, let's look at how
agents are built today and how the field advanced since I wrote [Large Language
Models at Work](https://vladris.com/llm-book/).

## LLMs @ Work Teardown

Here's the table of contents of my now quite stale book:

1. *A New Paradigm* - Mostly introducing what was then a very new and emerging
   field.
2. *Large Language Models* - Covering the basics - models, tokens, API calls
   and responses.
3. *Prompt Engineering* - Discussing prompt creation and tuning.
4. *Learning and Tuning* - Covering in-prompt, N-shot learning and fine tuning.
5. *Memory and Embeddings* - Going over context limitations and memory
   implementations.
6. *Interacting with External Systems* - About function calling.
7. *Planning* - Multi-turn execution of a plan - remember this was written before
   reasoning models and agents were the norm.
8. *Safety and Security* - Pretty self-explanatory.
9. *Frameworks* - Talking about LangChain and Semantic Kernel.
10. *Closing Thoughts* - Again, self-explanatory. GPT4 was the new hot model when
   I wrapped up the book.

Looking back at this, it's quite impressive how much has changes since.

Starting with Chapter 2, the correct term today is *foundational model*, the large models are multi-model, not limited to language.

Prompt engineering (Chapter 3) fell out of favor, the modern term is *context
engineering*, which is a superset of that: it's not only about the wording of
the prompt we give the model, but all the other things we stuff into its context
window - relevant instructions, documents, available tools - to produce the
optimal response.

My Chapter 4 was called *Learning and Tuning*. I think a better name would be
*In-context Learning vs Tuning.* This is still an interesting debate, where the
different approaches work better for different scenarios. To quickly summarize:
if you want your model to do *X*, do you craft a solid prompt or fine-tune it?
The tradeoff is there's only so much you can do with a prompt vs. fine-tuning is
expensive and you have to redo it for every new foundational model. An
interesting deciding factor is the complexity of *X*. How much domain knowledge
does *X* require? How divergent are your expected answers from what the model
tends to output?

*Memory* (Chapter 5) is a very interesting topic. For one, models today have much
larger context windows and they keep growing, so for a lot of small-scale
scenarios, this might be a non-issue. E.g. you can stuff a whole novel into the
context, something that wasn't possible two years ago. Industry leaders are now
offering memory built into their chat experiences, so I wouldn't be surprised if
in the near future memory gets abstracted away and handled underneath the API.
For the cases where we do have too much data to fit into the context, handling
this is now part of the *context engineering* discipline - determining what is
relevant, retrieving it, and putting it into the context.

*Interacting with External Systems* (Chapter 6), is the fastest moving. This is no
surprise, as model interactions with side effects produce the most value. When I
started writing the book, there was no standardized function calling in the
OpenAI API. You had to ~~prompt~~ context engineer the model into telling it
what it can do and give it some examples of JSON functions calls. Before I got
to Chapter 6, OpenAI introduced functions as part of the API. Nowadays MCP is
the standard, providing complex capabilities to models. I will talk more at
length about this below.

Chapter 7 was about *Planning*. Multi-turn execution was pretty novel two years
ago but today in underpins all agentic workflows.

*Safety and Security* (Chapter 8) remains a concerns. The attack surface only
grew. Two years ago, one of the biggest concerns was prompt leaking - an
attacker stealing your carefully crafted prompt. Today, with multi-turn
execution and MCP tools, the attack surface and opportunities for data exfil is
significantly larger.

As for *Frameworks* (Chapter 9), they kind of fell out of favor lately. As the
major players are pulling more capabilities within their APIs and models are
becoming smarter, generic frameworks are becoming maybe less popular.

As I wrote at the start of this post, if the field eventually slows down, there
will be plenty of opportunity to create frameworks capturing the latest patterns
and formalizing them into easy-to-use systems. But we're not there yet. After
this whirlwind review of how my last book is not so much relevant today, let me
talk a bit about the state of the art. This is going to be less of a *how to*,
more of a top of mind.

## Agent Engineering

The biggest leap over the past two years has been giving model-based solutions a
lot more autonomy. An *agent* is usually doing multi-turn work, with a plethora
of tools at its disposal. A key learning was that new models provide such giant
leaps in capabilities that today's scaffolding need to accommodate for that,
which we call *model-forward* engineering. I'll cover these and a few other
concepts below.

### MCP

The Model Context Protocol introduced by Anthropic provides a "standard" way of
capability providers to advertise their capabilities to a model and, vice-versa,
a common way for models to invoke tools. While MCP is not an official standard,
the fact that both Anthropic and OpenAI embraced it made it the de-facto
standard in the industry.

Whatever service you build, if it can *speak* MCP, agents can use it.

If you are building an agent, there's now a standard way to tell it what are all
the various tools it can use via MCP.

I expect very broad adoption of MCP. Most service will support this and you
would be missing out if you're building a service that wouldn't.

If you're building an agent, you'd want to supply its tools via MCP.

The part that I find most interesting is specialized agents exposing their
capabilities via MCP to other agents.

### Multi-Turn Workflows

In the world of agents, multi-turn is the norm. There are few "serious" jobs you
can achieve with a single model call. The model will either ask for a tool call,
revise its plan, or even prompt the user for input.

Tool use is the easiest to solve: you offer the model the buffet of MCP tools
available, it responds by describing a function invocation, your code runs the
function and gives it the response.

Plan revisions are more interesting: with a multi-turn agent, the agent should
usually starts by creating a to-do list, then go through it turn by turn. What
happens when new information causes the plan to change? What is the right
combination of prompt, tuning, context that will make the model realize "oh, I'm
going down the wrong path, let me step back and try something else"?

Prompting the user for input is another very interesting one. Here I am not
talking about permissions. I've been using a lot of coding agents which, for
security reasons, prompt you before running bash commands etc. I'm talking about
the model realizing it needs some more input from you, stopping, and asking you
a clarifying question. I've rarely seen this done well with existing agents, I
think the field is ripe for opportunity and I'm looking forward to advancements
here. As a concrete example, coding agents are still writing at a junior
engineer level. I would much prefer them to stop early and ask clarifying
questions rather than doing odd things to get things to work. I also realize
this is a very hard problem.

The bottom-line, as we're talking about agents, is you will probably need a
bunch of code to support long-running, multi-turn workflows.

### Tactical Fine-Tuning

Fine-tuning is taking a pre-trained model and teaching it some new stuff in the
form of prompts and expected replies. This works well for domain-specific tasks,
but it has some important trade-offs.

One trade-off is the fact that you'll have to fine-tune any new foundational
model you'll upgrade to. Fine-tuning is not cheap, it's an involved process, and
as new models come out quite often, there's an ongoing tax to pay to have a
bleeding-edge fine-tuned model ready to go.

An even bigger trade-off is a fine-tuned model will behave differently than an
"off-the-shelf" one. In a complex system, users might expect similar behavior
across surfaces but if one of the surfaces behaves differently the overall
experience will seem off. The way to mitigate this is to use *multi-agent
workflows*.

### Multi-Agent Workflows

The concept of *multi-agent workflows* is what got me to write this post. We
looked at MCP, multi-turn, fine-tuning - the combination of these is
mind-bending. With the state of the art, an ideal agent implementation has the
following ingredients:

* Specialized agents fine-tuned on specific tasks.
* Having these specialized agents implement the MCP protocol as servers.
* An "entry-point" agent that can leverage these other specialized agents via
  MCP.

This idea is what got me thinking of design patterns and what those mean in the
AI world. It's a very interesting horizontal scaling concept - rather than
having a one-size-fits-all solution, have a bunch of specialists ready and an
entry point that can leverage them as needed.

Coincidentally, I was procrastinating writing this blog post for a week or so as
Anthropic just announced Claude Code supports "subagents", one specializing in
code review, on specializing in debugging etc.

I'm positive we're entering the era of horizontal scaling where a solution will
involved a set of specialized agents rather than a "know-it-all".

### Model-Forward

This is a very interesting concept which means *How can we write software that
doesn't incur a tax when the new model comes out, rather it gets better as model
capabilities improve?*

The hard-learned lesson here was that, after building frameworks, scaffolding,
prompts, and sinking a ton of engineering cost into an AI-powered solution, when
the new model came out, it could do all (most?) of that stuff out-of-the-box, no
crutches needed. Model-forward design is about future-proofing and thinking a
move ahead.

Can we build solutions that will just get better as the models get better?
Fine-tuning is a controversial example of this - it's costly and different, and
maybe a new, smarter model can do whatever we want to achieve without
fine-tuning.

Predicting the future is impossible, so what *can* we do to be model-forward?
What we can do is offload more of the work to the model. The smarter it gets,
the more it can do for us. We can give it more tools, better context, and let it
do its thing.

The key point behind model-forward development is to focus less on complex
scaffolding around model workflows and more on giving the model the tools it
needs and letting it work. Rather than creating a complex, multi-step
orchestration, just hand the model the right context, MCPs available, and listen
to its response.

## Summary

In this post I looked at the analogy between AI model integration patterns and
the classic Design Patterns. I also did a teardown of my Large Language Models
at Work book, and looked at some of the emerging patterns:

* MCP as a standard.
* Multi-turn as the norm.
* Fine-tuning for specialized tasks.
* Multiple agents working together.
* Future-proofing with a model-forward approach.

No book for now, and not any time soon. I'll either write a book on how to use
AI when we reach stable state, or we pass the event horizon and AI will write
the book much better than I ever could. I've been refocusing on writing fiction
lately.
