# Notes on Documentation

I spent a bunch of time lately revamping some documentation and this got me
thinking. In terms of tooling, even state-of-the-art documentation pipelines
are missing some key features. This is also an area where we can directly apply
LLMs. In this post, I'll jot down some thoughts of how things could look like
in a more perfect world. Of course, here I'm referring to documentation
associated with software projects.

## Build from source

This first one isn't unheard of: documentation should be captured in source
control and generated from there as a static website. There are two major types
of documentation: API reference and *articles* that aren't tied to a specific
API.

API reference should be extracted from code comments. Different languages have
different levels of "official" support for this. C# has out-of-the-box XML
documentation (`///`), JavaScript has the non-standard but popular [JsDoc](https://jsdoc.app/)
etc.

Articles on the other hand should be written as stand-alone Markdown files.

A good documentation pipeline should support both. My team is using [DocFX](https://dotnet.github.io/docfx/)
to that effect, though TypeScript is not supported out-of-the-box and requires
some additional packages to set up.

## CI validation

Commenting APIs should be enforced via linter. We have tools like [StyleCop](https://github.com/StyleCop/StyleCop)
for C# and a [JsDoc plugin for eslint](https://www.npmjs.com/package/eslint-plugin-jsdoc)
for JavaScript. At the very least, all of the public API surface should be
documented. If you introduce a new public API without corresponding
documentation, this should cause a build break.

For technical documentation, many times articles also contain code samples.
These run the risk of getting out of sync with the actual code as the code
churns. In an ideal world, we should be able to associate a code snippet from
an article with a test that runs with the CI pipeline. Documentation might
skip scaffolding for clarity, so it's likely harder to simply attempt running
the exact code snippet. But we should have a way to pull the snipped into a
test that provides that scaffolding.

Alternately, enforce that running all snippets in an article in order works -
treat articles more like Jupyter notebooks, where the runtime maintains some
context, so if, for example, I import something in the first code snippet, the
import is available to subsequent code snippets.

The key thing is to have some way to validate at build time that all code
examples actually work and not allow breaking changes, even if the only thing
that breaks is documentation.

## Ownership

From my personal experience, documentation is usually treated as an
afterthought. From time to time there is a big push to update things, but it's
rare that everyone is constantly working towards improving docs.

Unless documentation reaches a critical mass of contributors to ensure
everything is kept in order, it's best to have clear ownership of each article.
Git history is not always the best for finding owners - sometimes the last
author is no longer with the team or with the company, or maybe last commits
just moved the file around or fixed typos.

This concern goes beyond documentation, in general I'd love to see an ownership
tracking system that can associate assets with people and is also org-chart
aware - so if an owner changes teams, this gets flagged and a new owner must be
provided.

## Inline fragments

While working on documentation, I noticed that for a large enough project, some
information tends to repeat across multiple articles. Maybe as part of a summary
on the front page, then again in an article covering some of the details, and
once more incidentally in a related article.

The problem is that if something changes and I only update one of the articles
(maybe I'm not aware of all the places this shows up), documentation can start
contradicting itself. This is something that is not part of the common Markdown
syntax but I'd love to have a way to inline a paragraph across multiple
documents to avoid this.

## Style guides

All documentation should include a style guide. Some guidelines encourage
writing for easier reading, so apply in most cases. For example:

* Avoid passive voice.
* Encourage diagrams and pictures vs. very long descriptions.

Some guidelines depend on the type of article. If you're documenting a design
decision, explain the reasoning and list other options considered and why these
weren't adopted. On the other hand, if you are writing a troubleshooting guide,
no need to explain the *why*, just what steps the reader needs to take.

Unfortunately I haven't seen a lot of such guides accompany projects. I wish
we had a set of industry standard ones to simply plug in, like we do with open
source licenses.

## Information architecture

In many cases, there is little effort put into structuring the documentation.
We start with `/docs` then as articles pile up, we create new subfolders
organically.

Much like we want some high-level design of a system, we should also require
a high-level design of the documentation. What are the key topics and
sub-sections? This doesn't even need to be reinvented for each new project,
I expect there's a handful of structures which can support most projects,
so much like style guides, it would be great to have these available
of-the-shelf.

### An alternative to hierarchy

I started this post talking about building documentation from source, which
naturally maps to articles being files organized in folders (categories). This
type of organization - categories and subcategories - works well up to a
certain volume of information.

At some point, it gets hard to figure out which subcategory something fits in:
it might fit just as well in multiple places. Here the folder categorization
breaks down: there is no clear hierarchy of nested folders in which to fit
everything.

At alternative to hierarchies are tags. Maintain a curated set of tags, then
tag each article with one or more tags. You can then browse by tag, but have
articles show up under multiple tags. This tends to work better with larger
volumes of information, but it's harder to map to a file and folder structure.

## AI

With the popularity of large language models, I see many applications
throughout the lifecycle:

### Authoring

Generative AI can help coauthor documentation. GitHub Copilot [already does
this](https://learn.microsoft.com/en-us/shows/introduction-to-github-copilot/how-to-write-documentation-with-copilot-suggestions-5-of-6).
As models get better and cheaper to run, I expect they will be more and more
involved in writing documentation.

### Reviewing and editing

Given a style guide, a model can review how closely a document adheres to it
and suggest changes to match the guide.

With a knowledge of the whole documentation, a model could also spot
contradictions (the problem I mentioned in the **Inline fragments** section).
This could be a step in the CI pipeline to ensure consistency.

A model could potentially also act as a reader and provide feedback on how
clear the documentation is.

### Retrieval

Most tools generating documentation from source provide very rudimentary search
capabilities. OpenAI offers text and code [embedding APIs](https://openai.com/blog/introducing-text-and-code-embeddings)
which enable semantic search and natural language querying. Using something
like this on documentation should make finding things much easier.

### Q&A

Models can also be used to answer questions, so instead of readers having to
search the docs for what they need, they can simply ask questions. A model
can provide answers based on the documentation (and the codebase). This takes
retrieval a step further: users can simply get their questions answered by a
model. In some cases articles might not even be needed, as the model can
explain in real time how the code is supposed to be used.

## Summary

I believe as of today, even the best tools available for documentation leave
room for improvement and large language models have the potential to radically
change the game.

In this post we looked at:

* The two main types of documentation: API reference and articles, which should
  both live in source control.
* API reference should be extracted from code comments.
* Articles should be written as Markdown.
* CI validation for documentation: enforce API documentation and ensure code
  samples still work.
* Ownership tracking: ensuring someone feels responsible for every piece of
  documentation.
* Inline fragments as a proposed solution to keep information in sync across
  multiple documents.
* Style guides for documentation to ensure consistent & readable articles.
* Information architecture to improve overall structure and navigation.
* Potential AI applications throughout the lifecycle:
  * Coauthoring documentation with generative AI.
  * Reviewing documentation and providing suggestions (for style, consistency,
    readability).
  * Finding the right documentation using embeddings.
  * Answering natural language questions.

Some of these features exist and some of these practices are adopted in some
projects, but most are not widely implemented. I'm curious to see how the
landscape will look like in a few years and how AIs will change the way we
learn and get our questions answered.
