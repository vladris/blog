# Notes on Authoring Tools

I always enjoyed coding, and I always enjoyed writing. Over time, I made a
bunch of tools for authors, which I used myself. I believe there's still a lot
of room to improve the authoring experience. Here is a short history of some
of these past projects.

## Tinkerer

About 12 years ago, I built [Tinkerer](https://github.com/vladris/tinkerer), a
now retired static website generator for blogs. This was back when Ruby was
extremely popular and many blogs were using [Jekyll](https://jekyllrb.com/) for
their blogs.

I was learning Python at the time, and figured building a similar blogging
engine on Python would be a good learning exercise. I created this on top of
[Sphinx](https://www.sphinx-doc.org/en/master/), which is a static website
builder used for documentation. This was pretty much what everyone in the Python
world was using at the time. Tinkerer added a set of extensions for Sphinx to
provide things like categories, metadata, RSS-feed generation, plug-in Google
Analytics, comments via Disqus etc. plus a command line interface to manage blog
posts and build the website.

It was a neat project, with several contributors adding fixes and features to
it. I used it for many years for this blog.

I ended up retiring it 2 years ago. The Sphinx project made some changes in
newer versions which broke some of the functionality. I pinned the Sphinx
version but that didn't feel like a good solution. Another drawback was that
Sphinx supported reStructuredText, which was very popular, at least in the
Python world, before Markdown took over the world. Today it seems Sphinx
extended support to Markdown but as far as I know this was not the case
when I decided to retire Tinkerer.

It has a good run of ten years, for a small learning project I did on the side.

## Baku

Of course, I wanted to continue building my blog with my own tools. I took the
learnings from Tinkerer and build [Baku](https://github.com/vladris/baku/). Baku
keeps the simple command line interface of Tinkerer, but builds from Markdown. I
also stripped it down to the features I found myself using most. Baku produces
an RSS feed, a chronological list of posts, "previous post" and "next post"
navigation, but not much else. I did away with categories, analytics, comments.

Some of these can be easily added if needed via the website template. Speaking
of templates, after being broken by Sphinx once, I wanted Baku to have minimal
dependencies. Sphinx uses [Jinja](https://jinja.palletsprojects.com/en/stable/)
as a templating language out of which it builds HTML. Templates contain HTML and
placeholders with a custom expression language which is evaluated at build time.
For Baku, I was able to replace this with [about 100 lines of code](https://github.com/vladris/baku/blob/main/baku/templating.py),
providing a rich enough language for my needs: `for` loops, conditionals,
variable access. I called this `VerySimpleTemplate`, and in fact ended up
reusing for another static website (that was not a blog).

I converted my old posts from reStructuredText to Markdown using
[Pandoc](https://pandoc.org/), and have been using it since. The blog you're
currently reading was built with Baku.

## WriteRT

10 years ago, the Surface tablet launched, with Windows RT and a new app store.
As another learning exercise, I built a small app for focused writing, very
much inspired by [iA Writer](https://ia.net/writer), which did not have a
Windows version back then.

I published [WriteRT](https://apps.microsoft.com/detail/9wzdncrdn98d) to the
app store. It fared pretty well for a project I put together during the holiday
break, while I was learning the new WinRT developer platform.

## Other Tools

Which brings us to today. Since building Tinkerer, I published quite a few blog
posts, a bunch of articles, and several books. I wish for a better tool for
long-form writing.

I used Word quite a bit, and not just because I've been part of the Office
organization for so many years. It is a good tool, but it is a multi-purpose
tools, not necessarily aimed at creative writing, long form writing and so on.
[Scrivener](https://www.literatureandlatte.com/scrivener/overview) has a lot of
great features, but I don't find its look and feel very inspiring. In fact, the
best designed editors I used over the past year or so are
[Bear](https://bear.app/) and [Obsidian](https://obsidian.md/). I love the
inline Markdown, where the markup disappears if not under the cursor. I love the
theming support, as I'm very particular about color schemes. I used [Solarized
Dark](https://ethanschoonover.com/solarized/) for a long time, then more
recently switched to [Tokyo Night](https://github.com/tokyo-night/tokyo-night-vscode-theme).
I've also toyed with [Nord](https://www.nordtheme.com/). Both Bear and Obsidian
are great apps, but they are meant for note-taking. Some of the organizing
features do come in handy for some long form writing, but they are also missing
features an author would like to have handy (for example, some revision tools).

I know there are many other tools out there, some of them I'm not covering as
I didn't find them interesting, others I might not know about, so please
consider this a very short and incomplete summary of the landscape.

For my latest writing projects, I've been using Obsidian. For now.

## Saturn9

Earlier this year, I decided to build something. I got <https://saturn9.studio/>,
and I put up a few explorations. I was originally considering building a text
editor from scratch, which I still think would be a very fun project, but a
text editor is a very complex piece of software. I might still do this at some
point in the future, as a learning exercise, but for now I settled on
[ProseMirror](https://prosemirror.net/), a great open-source editor with a
plug-in architecture.

In my spare time, I've been hacking away at a core experience. There's still a
bunch of missing features and a lot of polish that needs to happen, but I hope
to eventually bundle this into an Electron app and publish it as an app.

I want to start small, with an initial experience centered around
distraction-free writing. A retake on WriteRT, with better Markdown support,
a more polished experience, themes and such. Then build on top of that to add
features for what I see as the different modes for authoring:

* Inspiration and distraction-free writing - Have inspiration to strike and
  put it all down.
* Organizing - Managing outlines, chapters, notes, with easy navigation and
  retrieval.
* Revising - Annotating, proofreading, grammar and style checking and so on.
* Publishing - Exporting to various formats used for web publishing, ebooks,
  or print.

It's slow progress, as it is a side-project, but I have a few core pieces
working to some extent.

Here's a demo of (a subset of) Markdown support, automatically hiding markup and
only revealing it when the cursor is on the marked element:

<div id="demo1" class="editor-wrapper"></div>

This still has a ton of bugs to work through and polish - backspace over hidden
markup doesn't work well, new line in list should assume new list item, loading
Markdown documents needs work, formatting commands are not implemented and so
on. But I've been having fun learning about the complexities of text editors
and stitching this together.

Since I mentioned focused writing, here's a distraction-free experience in full
screen mode (and some inspiring placeholder text) based on the same editor. Just
start typing to see it work:

<div id="demo2" class="editor-wrapper"></div>

I also really wanted theming support, which is fairly easy to do on the web.
This is in fact mostly CSS, less editor core, but here's an example of theme
switching between Nord and Solarized:

<div>
  <label>
    <input type="radio" name="theme" value="theme1" checked>Nord
  </label>
  <label>
    <input type="radio" name="theme" value="theme2">Solarized
  </label>
  <div id="demo3" class="editor-wrapper"></div>
</div>

Authors also care a lot about word count. Below is an example of selection-aware
word counting. Try it out by selecting different parts of the text.

<div>
  <p id="word-counter"></p>
  <div id="demo4" class="editor-wrapper"></div>
</div>

<script src="bundle.js"></script>
<script src="d.js"></script>

Very early alpha, but some bits work. I'm planning on polishing this over the
following months into a first version of a full-featured editor.

### AI

Of course, I must touch on AI, since AI is everywhere nowadays. I actually have
some strong opinions on how large language models and creative writing
intersect.

As an author, asking AI to write something for me or rewrite something to be
more like this or more like that feels a lot like cheating. I want to use my own
voice. It's my writing, not the model's.

That said, LLMs bring some amazing super-powers to authors. Brainstorming is
one of them - idea generation for when you don't feel inspired. Another great
one is critique - a good model can gave you valuable feedback on strong points
and areas of improvement for your writing. Research is yet another one - I've
been asking ChatGPT questions about military ranks, slang, and so on for the
sci-fi book I'm working on.

I know a lot of editors are integrating AI features now, and I think most of
them are missing the mark on how to best empower authors. I want to build my
take on it, which I hopefully will in the fullness of time.

That said, I need a core authoring experience first, before looking into adding
AI features, so that's what I've been playing with so far. I'm optimistic of the
timing, and hope that by the time I have a solid authoring experience, models
will be commoditized enough to make the scenarios I'm thinking of lighting up
extremely cheap if not free.

Next year I will probably write a few blog posts on some of the interesting
challenges I encountered while working on this authoring tool.
