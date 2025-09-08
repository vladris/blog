# DevLog 5: Markdown and WYSIWYG

In this post I want to cover one of the foundational features of
[Flow](https://saturn9.studio/flow/): its Markdown handling. I will not show a
lot of code, because most of it is boring parsing code and special-case
handling.

Different editors tackle Markdown in different ways. Some support a subset of
Markdown formatting on insertion, but don't preserve any markup. For example,
typing a `*` would toggle Italics on then, once the user types the closing `*`,
not only are Italics toggled off, but both `*` markups are removed from the
document. This is the behavior of editors that don't natively support Markdown.

Other editors support Markdown natively, like some of the apps that inspired
Flow - [Obsidian](https://obsidian.md/) and [Bear](https://bear.app). The
problem with these apps is that Markdown has its own quirks which impact the
overall authoring experience. One simple example is handling of paragraphs: in
Markdown, a new paragraph starts after two newline characters. A single newline
doesn't affect the rendered document.

For Flow, I wanted to fully support Markdown, but also hide some of its quirks
from the user. If the user is not familiar with it at all, they should still be
able to intuitively use the app and have the best possible experience.

## Pre- and post- processing

Flow uses pure Markdown and can load and save `.md` files. To smoothen things
out, the first thing it does when loading a document is convert it into a series
of paragraphs by handling some of the whitespace particularities of Markdown.

It converts a piece of text like the following

```text
This is considered a single
paragraph in Markdown even though each
line ends with a newline character.

* This is a list item
* This is another list item
```

into something like

```text
<p>This is considered a single paragraph in Markdown even though each line ends with a newline character.</p>
<p>* This is a list item</p>
<p>* This is another list item</p>
```

The above is just to illustrate the type of transformation, we don't go straight
to `<p>` HTML elements, rather these are ProseMirror paragraphs.

The preprocessor is fairly simple, stripping single newlines where appropriate,
handling whitespace at the end of lines etc. It is also aware of lists,
codeblocks and block quotes, which have different rules. For example, a
codeblock is a single entity regardless of how many consecutive newlines appear
in its content.

The preprocessor runs when opening a new file. Conversely, the postprocessor
runs on save and converts the ProseMirror document back to Markdown by doubling
newlines where appropriate.

These are implemented by two functions:

```typescript
export function markdownToDoc(markdown: string): ProseMirrorNode {
    /* ... */
}

export function docToMarkdown(
    doc: ProseMirrorNode
): MarkdownSerializationResult {
    /* ... */
}
```

The `MarkdownSerializationResult` is defined as:

```typescript
export interface MarkdownSerializationResult {
    markdownText: string;
    positionMap: (processedPos: number) => number;
}
```

The first function takes a Markdown string as input and creates the ProseMirror document (returning the root node).

The serialization function takes the document and returns the Markdown as a
string. In addition, it also returns a mapping from position in the Markdown
string back into the document. That is used by the parser, which we'll look at
below. First, let's look at the internal document representation.

## Internal representation

ProseMirror allows for rich document schemas. In fact, the [ProseMirror Markdown
plugin](https://github.com/ProseMirror/prosemirror-markdown) can translate a
Markdown document into its HTML representation and vice-versa, where a paragraph
of text becomes a `<p>` element, a list becomes an `<ol>` or `<ul>` consisting
of multiple `<li>` elements, and so on.

I did not use this plugin because it strips the markup during conversion. I
wanted to leave all the markdown in the document. To achieve this, Flow uses a
very simple document schema consisting exclusively of paragraphs which translate
to `<p>` elements. All styling is done via decorations. I talked about
ProseMirror [decorations](https://prosemirror.net/docs/ref/#view.Decorations) in
previous posts, but to recap, at a high level, decorations add attributes to a
range of text. If my paragraph is `There are *italics* in this sentence`, we can
add a decoration from the starting `*` to the closing `*`. The rendered HTML
will look something like this `<p>There are <span>*italics*</span> in this
sentence`. The `<span>` will have whatever attributes we want to attach to it.

The document schema is just:

```typescript
export const docSchema = new Schema({
    nodes: {
        doc: { content: "block+" },
        paragraph: {
            group: "block",
            content: "inline*",
            attrs: {
                type: { default: "paragraph" },
                widgetId: { default: null },
            },
            toDOM(node) {
                return [
                    "p",
                    {
                        type: node.attrs.type,
                        widgetId: node.attrs.widgetId,
                    },
                    0,
                ];
            },
        },
        text: { group: "inline" },
    },
});
```

While the schema of Flow documents is simple, consisting of just paragraph
nodes, a lot happens at runtime to properly decorate the document.

## Markdown processing

Flow uses `markdown-it` internally to parse the Markdown document and decorate
it for rendering. There's quite a lot of parsing logic, and since this is one of
the core capabilities of the app, I have a lot of tests around it. Properly
parsing Markdown is not easy!

I mentioned above that `There are *istalics* in this sentence` roughly
translated to `<p>There are <span>*italics*</span> in this sentences</p>`. To be
more precise, this translates into `<p>There are <span class="em markup
hide-markup">*</span><span class="em">italics</span><span class="em markup
hide-markup">*</span> in this sentences</p>`.

The parser adds the `markup` and `hide-markup` decorations around the `*`s and
the `em` around the whole thing. ProseMirror maps these to the right spans on
render. We're also aware of the cursor position: if the cursor is within the
range of the decoration, we tag the markup with `show-markup` instead of
`hide-markup`.

Then everything is CSS: we simply get the `em` class to be italicized, `markup`
to be muted, `hide-markup` to be hidden etc. That's for the standard editor
mode. When we switch Flow to Source mode, both `hide-markup` and `show-markup`
are visible.

Here is where the `positionMap` comes in handy. Our ProseMirror-to-Markdown
conversion returns a `string` and a mapping. We briefly looked at this above:

```typescript
export interface MarkdownSerializationResult {
    markdownText: string;
    positionMap: (processedPos: number) => number;
}

export function docToMarkdown(doc: ProseMirrorNode) {
    /* ... */
}
```

The reason we need this mapping is we don't only convert the document from a
ProseMirror schema to a Markdown string on save, but also to parse it and add
decorations. When we parse the Markdown and find positions where we need to add
decorations, we need to map these back to positions in the ProseMirror
document. These aren't exactly identical.

### Positions

A quick side-note on positions: in ProseMirror, when indexing by position, we
need to account for non-text nodes (paragraphs in our case). A ProseMirrorNode
that is not a leaf text node has a "before" position and an "after" position.
This offsets things, so without maintaining a `positionMap` we end up indexing
the wrong document fragments. During serialization via `docToMarkdown()`, we not
only produce the Markdown string, but also track offsets to account for this.

### Lists

The story is simple for basic markup. It gets a bit more complex for things like
lists. Because I wanted to Flow to have a WYSIWYG experience, I'm using
single-line lists. That is, each list item gets converted to a separate
paragraph inside the document schema. We also want to preserve the markup, so
for an unordered list, if the cursor is on it, we want to show the `*` markup.
If the cursor is on a different line, then we want to hide the `*` markup but
show the usual • symbol.

For numbered lists, we also need to extract the number and make it available an
attribute for rendering, since ultimately lists don't translate to `<ol>`,
`<ul>`, and `<li>` elements, rather to `<p>` elements.

### Plugins

The Markdown parsing library Flow is using, `markdown-it`, has its own plugin
system, which allows us to extend the parser with custom syntax. For example,
underlying with `~` is done with a plugin.

### Widgets

Some elements, like code blocks and images, are represented as UI widgets in the
editor (unless in Source mode). The parser adds special decorations for these
such that, if widgets are available, we can completely hide the content and
instead inject a UI widget in the canvas.

I will cover widgets in detail in a future post, as they are not as interesting
from the Markdown parsing perspective, but do have a lot of code behind.

### Tables

Flow does not support tables yet. The [CommonMark](https://commonmark.org/) spec
does _not_ include tables. Tables are an extension introduced by the [GitHub
Flavored Markdown](https://github.github.com/gfm/) spec. I will probably end up
implementing support as specified by GitHub.

The reason I didn't make this part of the MVP is that tables are not critical
for creative writing. That said, as I'm using Flow to author blog posts like the
one you are reading now, I will need support for both tables and equations.

### Conclusion

I would've very much preferred not to get into the business of parsing Markdown,
as Markdown nowadays is quite complex, which makes custom parsing logic
error-prone. That said, I couldn't find a better solution for my requirements,
namely:

* Load and save Markdown.
* Provide a WYSIWYG authoring experience, which includes hiding/showing the
  markup as needed.

## Summary

In this post I covered how Flow implements a Markdown WYSIWYG authoring
experience:

* Pre- and post- processing the Markdown document into a simple ProseMirror
  schema.
* The document schema, which consists of paragraphs and decorations.
* A custom Markdown-parser based on `markdown-it` that parses the document and
  adds decorations.
* Lists, widgets etc. require special handling.

This post was written with Flow. Try it out [here](https://saturn9.studio/flow/).
