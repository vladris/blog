# DevLog 6: Extending Markdown

Markdown is ubiquitous nowadays. LLMs speak Markdown. Text editors embrace
Markdown. I've been working with it both on my text editor side-project and
on my main job. Markdown is great as a lightweight, easy-to-read markup
syntax but it has its limitations. Everyone ends up working around those
limitations in one way or another, which makes it difficult to have good
interoperability.

The reason for this is that Markdown was originally envisioned as a
text-to-html but ended up having to support a lot more *stuff*.

As I mentioned in [Markdown and WYSIWYG](https://vladris.com/blog/2025/09/08/devlog-5-markdown-and-wysiwyg.html),
the de facto standard for Markdown is [CommonMark](https://commonmark.org/).
CommonMark defines the basics - bold, italics, and so on. It also defines
lists, quotes, images. But it misses a lot of fairly common formatting
elements and entities. I called out tables as an example in my previous post.
Everyone is familiar with the table syntax, but this was introduced by GitHub
as part of their [GitHub Flavored Markdown](https://github.github.com/gfm/)
spec.

GFM also introduced task list items:

```markdown
- [x] Todo 1
- [ ] Todo 2
```

Commonly understood by parsers but not "standard". Same with strikethrough.
This would be the `~~` markers, as in `~~struck~~`.

## Two Ways To Extend

### Custom Syntax

One way the syntax gets extended is with custom markers. The above strikethrough
is an example. There's also underline, which is commonly a single `~`, as in
`~underlined~`.

Footnotes are another common extension, where a footnote is specified inline via
`[^1]` and, at the bottom of the document, `[^1]: <Footnote content>`.

Math is another one, when we want to render some LaTeX-syntax math using a
library like MathJax. Inline math ends up between `$` markers and blocks end up
between `$$` markers.

Highlights are written with `==`, as in `==highlighted==`.

### Tags

The other way to extend Markdown is via HTML tags. Markdown allows embedded
tags which are pass-through in the parser.

If we want subscript, we can write, for example, `H<sub>2</sub>O`. A well-behaved
parser will leave `<sub>` and `</sub>` in there as tags and let the HTML
renderer take care of them.

We don't have to limit ourselves to standard tags though, we can define our own
for the scenarios we want to support. This is common especially in interactive
LLM chat scenarios. The models speak some version of "standard" or commonly
accepted Markdown. If we want to enhance that to improve the end user
experience, say replace an email with a name and profile picture, we can use
middleware to inject a `<custom-person>`-`</custom-person>` tag pair and have
the client interpret that.

### Tradeoffs

The first extension mechanism covered keeps true to the spirit of Markdown. You
can read the plain text and "see" `==highlighted==` is a highlight. Tags are a
bit harder to parse with our eyes.

On the flip side, tags allows us to add rich custom extensions (think additional
attributes on the tag) while keeping the document standard-conforming. Any
parser will notice there's some custom *stuff* there, and skip over it.

## Rich Text Editing

Both of the above work to a certain extent but neither is quite good enough for
interoperability between editors.

As I've been working on my text editor side-project, I'm reaching a point where
I want to go beyond the basics and implement some common editor features that
do not have a direct Markdown representation.

Up to this point, the extensions my editor supports are quite common: underline,
strikethrough, highlights. Now it gets more interesting. I want to add support
for multiple highlight colors.

What [Bear](https://bear.app/) does, which I ended up adopting, is using a block
color emoji to define the highlight color, like `==🟩text==` to signify a green
highlight. This looks very neat and you can "read" it without a parser to
understand what it does.

Another option is to introduce a custom tag, like `<highlight color="green">`.

The question is, at what point does a Markdown document stop being Markdown? If
we load the custom markup in a different Markdown editor, the most likely thing
to happen is for the `==🟩text==` to show up as a yellow highlighted "🟩text".
If we use the tags instead, a good editor will hide the tags, and leave you with
"text". By Markdown editor here I don't mean a plain-text editor, I mean
something like the app I'm building or any similar solution.

I ended up implementing the Bear solution for this, though as far as I know no
other editor support this syntax.

Highlights are a relatively easy problem. Here's a more complex one: comments.
A good editor should support comments, which would anchor on some text and
contain additional text rendered outside the flow of the document. How would we
represent comments in Markdown?

There's the custom syntax approach, like `[this is text]{and this is a
comment}`. Or maybe something like footnotes, which is another common extension.
We can add a `[#comment]` inline and store it at the end of the doc as
`[#comment]: ...`.

Or use tags. `<comment text="this is the comment">and this is the
text</comment>`. Or as a reference `<comment refid="1">this is the
text</comment>` and somewhere else have the

```markdown
<comments>
    <comment refid="1">this is the comment</comment>
</comments>
```

All of these are viable options but it should be pretty clear they all diverge
quite far from Markdown. Maybe the first option, with some custom bracket
combination, is the most Markdown-ish. I haven't yet decided how I will
represent comments.

This begs the question: is Markdown really the best format to use? An option
would be to switch my editor to a custom format that can easily support all
the features I want. This can make advanced stuff like comments easy to support
and get rid of some of the quirks Markdown has, for example the double newline
required to create a new paragraph. This would be an option but, as I said at
the beginning, Markdown is ubiquitous. I do want files created with my editor
to be easily understood by reading them, opening them in another text editor,
or sending them to an LLM.

So inevitably, the more features I add, the further I diverge from any common
Markdown implementation. My app's parser/renderer will be the only ones that can
properly interpret any document created with my app.

## Solutions

The pragmatic solution I will end up implementing, regardless of what flavor I
end up choosing to represent comments, is an "Export to common Markdown"
feature. Better be explicit about it. Strip custom markup/tags and, while losing
fidelity, provide an easy way to get the document closer to the "standard".

I keep putting "standard" in quotes because there is no real standard. It's
a combination of CommonMark, GFM, support for footnotes and math etc.

I think this is a good compromise between supporting all the features I want,
having a native document format that can still be read as plain-text/by an LLM,
and also providing a "standard" representation of it.

That said, I do think Markdown would benefit from a well-defined way to add
custom extensions. The format is so successful, it moved far away from *plain
text and HTML*. Dozens of text editors embraced it, AI chat embraced it, and
every endpoint brought its own additions.

The HTML tag pass-through was a good solution for the initial intended purpose,
but it feels like a crutch as an extensibility point. What would work better
would be a syntax akin to `(:<extension>: text)`. This would make it explicit
to a parser that a custom extension to the syntax is being used and let it
handle it as appropriate: discard it and keep the text only, delegate to the
extension if available, replace it with a placeholder etc. Not very different
from the HTML tags, but unlike HTML tags, this wouldn't be simply pass-through,
it would have semantic meaning to the parser. Something like this would also
make it easier for both humans and AI to reason over a document, even when the
extension interpretation is missing.

The proposed syntax is just an example, it doesn't have to be this, it just has
to be something everyone agrees on that meets the requirements.

With this, we would represent a green highlight as `(:highlight green: text)`.
Easy to read as plain text and an editor missing the extension could just render
`text`. Maybe even have some fallback to hint to the user the text is marked
but the markup is unsupported. A comment could be written as `(:comment "this is
the comment": text)`.

HTML tags would be used as originally intended, for custom HTML markup and not
to add features. LLMs would be able to clearly distinguish between custom
extensions and tags as the distinction would be explicit in the syntax rather
than inferred.

The Markdown ecosystem is extremely fragmented as of today and without an
agreed-upon solution to bring things back together it will just continue to
diverge. Custom notations like `~underline~` and `==🟩green==` will keep being
invented and supported inconsistently, while custom tags will continue to
provide non-portable extensibility. I wish we had a better way of doing this
and I'm worried we're too far down the road by now to retrofit it in.
