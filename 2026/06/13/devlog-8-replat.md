# DevLog 8: Replat

I had a couple of alternative titles for this post. *Side-quests.* Or *Hold my
beer…* I’ll explain.

I recently shipped the first version of [Longhand on the MacOS App Store](https://apps.apple.com/us/app/longhand-by-saturn9/id6761262924?mt=12).
It’s a solid minimum viable product. I already have a couple of bug fixes and a
handful of new features queued up for the next minor release. I also have a
couple of larger features planned. For example, I want to add support for
comments and change suggestions. Then implement a new *Review* mode where
instead of authoring one can highlight, add comments etc. Longhand already
supports multiple colors of highlighters but I think providing a dedicated mode
for review will be a neat feature (based on my personal workflow). Another
feature I want to provide is note pinning - currently one can navigate between
the manuscript and notes using the navigator, but while editing a specific
chapter, I want users to be able to quickly peek at the notes relevant to that.
Also export to Word in the standard manuscript format. A lot more…

Instead of picking these up, I had the great idea of ripping out markdown-it and
ProseMirror from my project and replat on a custom stack.

This sounds a bit crazy since *why would you reinvent the wheel* and *if it
ain’t broke don’t fix it* but I am trying to test the hypothesis that code is
cheap in the age of agentic coding. Of course, both markdown-it and
ProseMirror have years of bug fixes and corner cases, so there’s that case to
make against a replat. I do want to give it a try though, maybe just as an
experiment.

What prompted this? It takes time to understand the problem space enough to
identify the right solution. This happened to me with Flow and Longhand.
Longhand was always the application I had in mind: a feature-rich editor for
authors. Flow was a stepping stone that was much easier to ship while also
helping me understand the constrains.

Flow was built as an Electron app, with the goal of supporting both MacOS and
Windows with a single codebase. I learned that Electron is a hog (I ranted about
this before), since it bundles all of chromium AND it doesn’t work on mobile.
That made me to switch to Tauri for Longhand, which is a lot lighter and has
support for iPadOS and iOS. I will eventually get Longhand on all of these
platforms.

## Motivation

So what’s wrong with markdown-it and ProseMirror? Nothing wrong with them,
they’re just not a great fit for how I’m using them. They are, though, the
*best* fit. So I either stick with them and work around the quirks or build my
own. The editor I built is, at its core, a very very fancy plain-text editor.
The underlying format is a dialect of Markdown. One of the key features I built,
the “hiding the markup unless cursor is around it”, best exemplifies this.

### ProseMirror model

The way ProseMirror is designed to work is for you to describe a rich document
schema, which can translate to HTML: nodes that map 1:1 with `<h1>` and `<h2>`
tags, with `<ol>` and `<ul>` tags, to `<b>`, `<i>` tags etc. The library does a
great job of keeping the internal document representation in sync with the
`contenteditable` view. The plugin model allows adding decorations to the
rendered document, which translates to `<span>` tags with HTML attributes.
There’s even a Markdown plugin for ProseMirror that can serialize/deserialize
Markdown into the ProseMirror document model.

Unfortunately this doesn’t work for my editor(s). The problem is that Flow (or
Longhand) doesn’t need to convert, say `**X**` markup into a `bold` node with
text `”x”` rendered as `<b>X</b>`. Flow needs the full `**X**`rendered, so we
can peek at the markup when the cursor is there. We just add decorations to it,
so we can make the “X” bold and hide or show the “\*\*” depending on the current
selection. That makes my document schema very simple: just blocks of text.
Styling happens based on decorations which are applied over the simple schema
via plugins.

Same mismatch with widgets. While I want a code block widget in edit mode, in
source mode I want to render the raw Markdown text. I ended up having to build a
registry mapping widgets to document positions which can’t properly garbage
collect. This is a known bug I had since Flow v1 which I haven’t prioritized
because it only happens when certain thrashing occurs but it’s another indicator
that the use-cases for what I’m trying to build don’t quite match what
ProseMirror offers.

ProseMirror also relies heavily on `contenteditable` for browser support. More
modern editors seem to move away from that to a different model handling input
via a hidden `<textarea>` and handling the HTML rendering themselves.

And my own pet peeve: ProseMirror position math is annoying - a node ends up
being the length of its content +2, which makes my code littered with weird +1s
and +2s.

## markdown-it model

markdown-it is meant to be a robust streaming parser that reliably converts
Markdown to HTML. The way I’ve been using it is to parse the text blocks and
generate decorations via ProseMirror. For this case, I only need the front-end
of the parser. I’m not outputting any HTML, I’m just figuring out where to add
decorations.

markdown-it, much like ProseMirror, is a Swiss Army knife of Markdown
parsing, with seams everywhere and great plugin support. I think I need
something a bit different: a Markdown parser that outputs the token stream I
need to consume in my editor(s). I also had a feeling a solution built
specifically for my use-case can be more performant than a Swiss Army knife.

The library worked for my scenario, but I only used the front end. I needed the
token stream to produce the decorations for ProseMirror. The end-to-end solution
involved a lot of position math, to reconcile the parser’s logic with
ProseMirror’s extra positions per node.

## Replat

Given the above, I made a bet on AI capabilities being good enough to support
such a replat without sinking months of work into it. I think the bet paid off.

### markoffset

First, I wanted a fast Markdown parser (actually front-end) that is *fast* and
standard-compliant. I now have

[markoffset](https://github.com/saturn9studio/markoffset), which I’m open
sourcing. The library is 100% plug-in driven, by design I wanted even the basic
syntax markup for bold (`**`) and italics (`*`) to be a plug-in and keep the
core parser clean. It benchmarks faster than markdown-it on my tests.

It passes the full CommonMark and GitHub Flavored Markdown set of tests.

The output of markoffset is the array of token objects with offset and length,
which I can directly plug into my editor.

markoffset also supports incremental parsing, which makes it even better suited
to integration in an editor. It can smartly re-parse only the part of the text
that was modified between edits, without requiring a full re-parse on document
change.

### scribeframe

Second, I wanted an editor that more closely matches the document model I’m
working with: a list of paragraphs consisting of Markdown text and decorations +
widgets.

Modern web editors do not use ProseMirror’s `contenteditable` mapping to HTML
and back. The modern approach uses a hidden basic input text area and the editor
canvas is painted by the editor rather than the browser, including simulating
cursor, selection etc.

Both Flow and Longhand rely on the same simple document schema - paragraphs and
decorated spans. Introducing
[scribeframe](https://github.com/saturn9studio/scribeframe). scribeframe takes
the good parts from ProseMirror - immutable document model, transaction-based
editing, plug-ins etc. It also ditches the legacy `contenteditable` approach to
editing and provides better widget support (see the leak issue I mentioned
above).

The “frame” in the OSS library‘s name is because it is generic enough to be
potentially used in other scenarios. “Scribe” is the internal editor I’ll be
using for Flow and Longhand, which adds the app-specific logic.

### Current status

At the moment I’m dogfooding a version of Flow that is built on Tauri, with
markoffset and scribeframe replacing the legacy editor stack. Overall it matches
the legacy Flow in functionality and sets me up to more easily add additional
features.

I will be releasing Flow v2 soon, then focus on swapping out the core editor for
Longhand. I’m still quite happy with the way I approached this, with a simple
editor to prove things out and a more complex editor that can benefit from
changes.

A few thoughts on agentic coding: I would’ve never embarked on such a replat
effort if it weren’t for this latest generation of AI tools. At the same time,
there are still gotchas with it and the adage that the last 10% takes up 90%
still holds. AI had no problem implementing a solid Mardown parser that passes
the full CM and GFM test suite and implementing a core editor from scratch.
Getting the Flow experience to 1:1 parity was an uphill battle though. A lot of
UI glitches, special-cases the AI missed, and so on. I had the agent solemnly
declare several times parity is reached only to discover more issues in the
reimplementation. That’s OK though, such an ambition rewrite would’ve taken
months and I managed to condense it in two weeks. For all intents and purposes,
we are *there* in terms of AI capability.

I will be releasing Flow v2 soon. This will be, for all intents a purposes, a
full rewrite replacing Electron with Tauri, ProseMirror with scribeframe,
markdown-it with markoffset. Then I will port the editor changes to Longhand.
Longhand is already a Tauria app and, with the new editor proven in Flow, I
expect it will be a much easier lift.
