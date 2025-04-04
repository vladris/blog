# DevLog 2: Formatting

One of the more interesting (and generally applicable) things I had to build for
[Flow](https://saturn9.studio/flow/) is the commanding infrastructure,
especially as it pertains to formatting. This entails going from the app UI all
the way down to the editor and vice-versa.

What I mean by this is, the user can apply a command like *toggle bold on the
current selection* in multiple ways: using the `Cmd+B` keyboard shortcut, or via
the menu bar, going to `Format -> Bold`, or via the context menu. I did not
implement a toolbar for Flow to keep the UI clean, but that could also be an
option.

Not only that, the menu and context menu should show a checkmark next to the
`Bold` command if the current selection covers only bold text. Similarly, a
toolbar button would show as pressed when the current selection covers only bold
text. This is standard behavior for a text editor.

I call this the *commanding infrastructure*: a way for the editor to expose a
set of commands like *toggle bold on the current selection*, including any
additional state required (like whether `Bold` should show a checkmark/pressed
button or not; keyboard shortcut etc.). This infrastructure needs to ensure all
the different UI entry points are views on the same state, and all invoke the
same editor behavior. This state needs to propagate from the editor to the UI
layer, since only the editor "knows" whether the text the user is currently on
is bold or not. Then the different UI entry points need a way to tell the editor
to make some change, e.g. *toggle bold on the current selection*.

An instruction like the above does *formatting*. Formatting includes changing
the content by applying properties like bold, italic, underline etc., turning a
paragraph into a header, a blockquote, a list item, and so on. There is quite a
lot of complexity to even something simple like *toggle bold on the current
selection*. I'll focus on this complexity in this blog post and I will cover
commanding in the next post.

## Bolding text

Since Flow uses Markdown, making some text bold means adding `**` markers around
it. Conversely, removing bold means removing the `**` surrounding markers.

This sounds pretty straight-forward, but there are many special cases that a
well-behaved editor needs to handle properly:

### No selection

If the user doesn't have a selection, just the cursor:

* If the cursor is not on a word, rather on whitespace, toggling bold should add
  `**` before and `**` after the cursor.
* If the cursor is on a word that is not bold, toggling bold should mark that word as bold.
* If the cursor is on a word that is bold, toggling bold should, depending on
  the editor, remove bold only from that word or for from the whole span. If the
  cursor is somewhere over `sentence` in `**This sentence is bold**`, depending
  on the editor you might end up with `**This** sentence **is bold**` or `This
  sentence is bold`.

### Selection spanning bold/not bold

If all the selected text is bold, then toggling it should remove the bold
markup. Otherwise, if at least some of the selected text is not bold, toggling
should make it all bold.

Selecting `**Some of this** text is bold` and applying bold to it should yield
`**Some of this text is bold**`, extending the bold markup to cover the whole
span.

What about removing bold? I've seen different editors also handle this
differently. If `[` and `]` denote the selection, what should happen when the
user toggles bold on `**Some of [this text] is bold**`? Some editors yield `Some
of this text is bold`, removing the whole markup, while others remove it only
from the selection, so you end up with `**Some of** this text **is bold**`.

Note we also need to be smart about cleaning up markup. If we toggle bold on
`[**Some of** this text **is bold**]`, we could naively do something like
`**Some of** **this text** **is bold**` but instead we should account for this
and end up with a cleaner `**Some of this text is bold**`.

### Selection spanning paragraphs

Markdown markup like `**` cannot span multiple paragraphs. If the user selects
multiple paragraphs, we need to apply the same logic to each paragraph in turn.
We can just add `**` at the beginning and end of the selection, rather we need
to add a `**` at the end of each selected paragraph except the last one in the
selection and add a `**` at the beginning of each selected paragraph except the
first on in the selection.

```text
This is a [multiple

paragraph text with selection

spanning multiple paragraphs] too
```

Toggling bold here should yield

```text
This is a **multiple**

**paragraph text with selection**

**spanning multiple paragraphs** too
```

We also need to take into account whether all the text in the selection, across
paragraphs, is bold or not, so we know whether you need to toggle it on or off.

For example:

```text
This is a [**multiple**

**paragraph** text **with selection**

**spanning multiple paragraphs**] too
```

If `[` and `]` denote the selection, applying bold to this should leave the
first and last paragraphs unchanged, since the selection in both is fully over
bold text. The only change should happen in the middle paragraph, which should
be fully bolded:

```text
This is a **multiple**

**paragraph text with selection**

**spanning multiple paragraphs** too
```

These are all various situations we need to account for when implementing
formatting.

In fact, formatting is an area where I have the most unit test coverage and I
keep adding tests as I uncover various cases.

## ProseMirror implementation

I'll cover how I implemented formatting. My editor is built on top of
[ProseMirror](https://prosemirror.net/), and I will not go to deep into
explaining how ProseMirror itself works. For that, refer to the
[documentation](https://prosemirror.net/docs/).

### Ranges and Decorations

Formatting is based on *Ranges* and *Decorations*. A *Range* simply defines a
range within the document, with `from` a `to` locations:

```typescript
type Range = {
    from: number;
    to: number;
}
```

*Decoration* is a ProseMirror concept[^1]. A plugin[^2] can add decorations to
the document, which are usually consumed by the view. A decoration contains
`from` and `to` positions in the document, a `spec` of type `any` to store
arbitrary information, and a set of *decoration attributes* which translate into
DOM node attributes when the document gets rendered. For example, if we select a
range in `"this **is a** string"`, say the `**is a**` part and set as attribute
`class` as `"bold"`, it will translate this as HTML into `this <span
class="bold">**is a**</span> string`.

I won't cover how decorations are generated in Flow in this post. I have a
Markdown plugin that parses the document and generates the appropriate
decorations. That's a topic for another post. For formatting, we simply consume
the produced decorations. One of the core functions fetches the decorations
produced by the Markdown plugin:

```typescript
export function getDecorations(state: EditorState): DecorationSet {
    const plugin = state.plugins.find(
        (plugin) => plugin.spec.key === markdownPluginKey
    );
    const pluginState = plugin?.getState(state);
    return pluginState?.decorations || DecorationSet.empty;
}
```

This is very ProseMirror-specific: we take an `EditorState` which represents the
state of the document, retrieve the Markdown plugin, get its state, then fetch
the decorations from the state.

Now that we can access the full set of decorations, we need a few more helper
functions to deal with selection ranges:

```typescript
export function isRangeDecorated(
    state: EditorState,
    className: string,
    from?: number,
    to?: number
): boolean {
    if (!from || !to) {
        from = state.selection.from;
        to = state.selection.to;
    }

    for (const decoration of getDecorations(state).find(from, to)) {
        if ((decoration as any).type?.attrs?.class === className) {
            return true;
        }
    }
    return false;
}
```

This function helps us check whether the given range is decorated with the given
class name. A decoration can have any attributes on it, but Flow uses class
names to tag things as bold, italic, etc. This functions tells us whether the
given class name appears anywhere within the selection. In other words, this
returns `true` if any part of the range is decorated.

It will return `true` if we query for `bold` and the selection range yields
`this **is a** string` since `**is a**` should have the decoration.

We also need a function that tells us whether a range is fully covered:

```typescript
export function isRangeCovered(
    state: EditorState,
    className: string,
    from?: number,
    to?: number
): boolean {
    if (!from || !to) {
        from = state.selection.from;
        to = state.selection.to;
    }

    for (const decoration of getDecorations(state).find(from, to)) {
        if ((decoration as any).type?.attrs?.class !== className) {
            continue;
        }

        if (decoration.from <= from && decoration.to >= to) {
            return true;
        }
    }
    return false;
}
```

This returns `true` if a decoration covers the full range. This will return
`false` for `bold` and `this **is a** string`, since the bold decoration only
covers `**is a**` rather than the whole range.

Now here is where things get a bit more interesting: a selection range can span
multiple paragraphs, but Markdown markup cannot. As we saw above, we could have
a selection that spans the following 3 paragraphs:

```text
This is a **multiple**

**paragraph text with selection**

**spanning multiple paragraphs** too
```

Note that all text starting from `**multiple**` on the first paragraph all the
way down to `paragraphs**` on the last line is bold, but we need some markup on
each line because of how Markdown works. In other words, we need to treat each
paragraph separately. The following function generates an array of ranges from
the current selection:

```typescript
export function getRangesFromSelection(state: EditorState) {
    const { from, to } = state.selection;
    const ranges: Range[] = [];

    state.doc.nodesBetween(from, to, (node, pos) => {
        if (node.type.name === "paragraph") {
            const paragraphStart = pos + 1;
            const paragraphEnd = pos + node.nodeSize - 1;

            const rangeFrom = Math.max(from, paragraphStart);
            const rangeTo = Math.min(to, paragraphEnd);

            ranges.push({
                from: rangeFrom,
                to: rangeTo,
            });
        }
    });

    return ranges;
}
```

We first get the `from` and `to` of the selection from the editor state. We then
iterate through each document node between `from` and `to`. For nodes of type
`paragraph`, we get the adjusted start and end positions (ProseMirror inserts an
additional position before and after each node, which we need to account for),
and determine the start and end of the selection range within this paragraph.

This function effectively turns a single `from`-`to` selection into several
ranges, one for each paragraph.

We already saw the `isRangeDecorated` and `isRangeCovered()` functions. I
implemented a couple of additional helpers, `areAllRangesDecorate()` and
`areAllRangesCovered()` which simply check whether all of the ranges returned by
`getRangesFromSelection` are decorated or covered by the given markup. The
implementation is trivial.

With these pieces in place, we can look at how Flow toggles formatting for the
current user selection.

### Toggling Formatting

Let's start with the core formatting functions, `markRange()` and
`unmarkRange()`.

`markRange()` takes a selection range and ensures it gets fully covered by the
given markup. For example, if we take the `this is a string` range, running
`markRange()` on it with the `**` markup will yield `**this is a string**`. But
that's not all! If instead we run it on `this **is a** string`, it will also
yield `**this is a string**`. Note we are effectively selecting text that is
partially bolded, and we turn it all bold - that entails not only adding `**` at
both ends, but also removing the inner-markup.

More than that, we also need to handle a case like the following (where `[]`
represents the user selection): `this **is [a** string]`. Making this selection
bold should yield `this **is a string**`. Not only we remove the inner markup,
we also need to be smart enough to know we already have bold markup before the
selection starts, so we shouldn't prepend `**`.

Here is the full implementation of `markRange()`:

```typescript
function markRange(
    state: EditorState,
    tr: Transaction,
    className: string,
    markup: string,
    range: Range
) {
    const markupLocations: number[] = [];
    let leadingMarkup: boolean = false;
    let trailingMarkup: boolean = false;

    for (const markupRange of getMarkupRangesForRange(
        state,
        className,
        range
    )) {
        if (markupRange.from < range.from) {
            leadingMarkup = true;
        } else {
            markupLocations.push(markupRange.from);
        }

        if (markupRange.to > range.to) {
            trailingMarkup = true;
        } else {
            markupLocations.push(markupRange.to - markup.length);
        }
    }

    tr = removeMarkupAtLocations(state, tr, markup, markupLocations);

    if (!leadingMarkup) {
        tr = insertMarkupAtPos(tr, markup, tr.mapping.map(range.from));
    }

    if (!trailingMarkup) {
        tr = insertMarkupAtPos(tr, markup, tr.mapping.map(range.to));
    }

    return tr;
}
```

This leverages several helper functions which I won't walk through in the
interest of time, but will explain what each does.

The function takes the following arguments: an editor state, a *transaction*,
the class name for the formatting as given by the decorations produced by the
Markdown plugin (e.g. `bold` for bold), the markup itself (e.g. `**` for bold),
and a range.

I haven't talked about transactions yet. Transactions is how ProseMirror handles
updates to the document state[^3]. Any change to the document needs to come in
the form of a transaction. A transaction can accumulate several operations
before being applied to the document. This architecture ensures consistency of
the editor state: if the state changes for whatever reason between the time a
transaction was created and the time it is dispatched, it will fail. This saves
us from dealing with issues like selection changing *while* we're trying to do
something to it which would result in unexpected results. Since all state
mutations come via transactions, ProseMirror will know when it needs to re-run
plugins - for example the Markdown plugin needs to run whenever the content of
the document changes to ensure our decorations are up to date with the text.
That's why all our formatting functions take a transaction as argument, to which
they add operations. To apply a transaction to the document, we need a
*dispatch* function which ProseMirror provides.

Back to `markRange()`, we first call `getMarkupRangesForRange()`. This is a
helper function that returns all the `from`-`to` ranges of all decorations with
the given class name intersecting our `range`. The *intersecting* part is
important: if can return ranges that partially overlap the range, fully cover
it, or are within it. I won't go over the implementation but it is
straightforward - it simply takes this data from the decorations produced by the
Markdown plugin.

For reach of the returned markup ranges, if the markup range starts outside our
range, we mark `leadingMarkup` as `true`, otherwise we keep track of the start
location of this markup range:

```typescript
…
if (markupRange.from < range.from) {
    leadingMarkup = true;
} else {
    markupLocations.push(markupRange.from);
}
…
```

We do the same for the end of the markup ranges:

```typescript
…
if (markupRange.to > range.to) {
    trailingMarkup = true;
} else {
    markupLocations.push(markupRange.to - markup.length);
}
…
```

Now we should have the locations of all markup inside our range, for example the
locations of all `**` inside the range we're trying to turn bold. We should also
know whether we have `leadingMarkup` and/or `trailingMarkup`, that is whether
there already is a leading `**` before or a trailing `**` after our range.

The next step is to remove all markup inside the selection - as a reminder, if
the user selects `this **is a** string` and wants to turn all of it bold, we
need to get rid of the markup around `is a` and the end result should be `**this
is a string**`. Since we already have all markup locations, `markRange()` calls
another helper function, `removeMarkupAtLocations()`:

```typescript
…
tr = removeMarkupAtLocations(state, tr, markup, markupLocations);
…
```

This helper function takes the editor state, a transaction, the markup we're
trying to remove and where it is located in the selection, then proceeds to
remove it. I won't cover the implementation to keep things short. Finally, in
case we don't have leading and/or trailing markup, we need to insert it:

```typescript
…
if (!leadingMarkup) {
    tr = insertMarkupAtPos(tr, markup, tr.mapping.map(range.from));
}

if (!trailingMarkup) {
    tr = insertMarkupAtPos(tr, markup, tr.mapping.map(range.to));
}
…
```

And that's it. The transaction we end up with contains all required edits to
properly mark the given range.

The inverse of this is `unmarkRange()`:

```typescript
function unmarkRange(
    state: EditorState,
    tr: Transaction,
    className: string,
    markup: string,
    range: Range
) {
    const markupLocations: number[] = [];
    for (const markupRange of getMarkupRangesForRange(
        state,
        className,
        range
    )) {
        markupLocations.push(markupRange.from);
        markupLocations.push(markupRange.to - markup.length);
    }

    return removeMarkupAtLocations(state, tr, markup, markupLocations);
}
```

This function is simpler than mark range. It also relies on
`getMarkupRangesForRange()` and stores all markup locations. As a reminder,
`getMarkupRangesForRange()` will include markups that intersect the range, so
even if the selection is inside a bolded range, like `**This [is a] string**`,
we would still get the locations of the two `**` markups.

Once we have these, we simply remove all markup by calling
`removeMarkupAtLocations()`. It turns out that removing markup is easier that
adding it.

The user issues the same command, *toggle bold*, and we need to figure out
whether we should add or remove the markup. We do this by relying on
`areAllRangesCovered()` which we described above. The `toggleDecoration()`
function takes an editor state, a dispatch function, a class name (like `bold`)
and a markup (like `**`). The function begins with:

```typescript
function toggleDecoration(
    state: EditorState,
    dispatch: (tr: Transaction) => void,
    className: string,
    markup: string
) {
    const ranges = getRangesFromSelection(state);
    const shouldToggleOn = !areAllRangesCovered(state, ranges, className);
…
```

We first get the user selection from the state and map it to one or more ranges
(to account for selections that span paragraphs). Next, if at least one of the
selected ranges is not fully covered by the relevant decoration (e.g. `bold`),
then we know we should toggle bold *on* across all ranges. Otherwise, if all
ranges are fully covered, we need to toggle bold *off*.

There's a bit more logic in this function which I won't cover that handles the
special case of not having a selection. If we only have a cursor, the expected
behavior is as follows: if the cursor is inside a marked range, e.g. inside a
bold range, toggling bold should remove the bold markup of that range. Note this
could be a word, a sentence, or even a whole paragraph. On the other hand, if
the cursor is not inside a marked range, the expected behavior is to apply the
markup just on the word the cursor is on. Try it out in your favorite editor, it
should behave like this!

One more special case: if the cursor is not on a word, rather on whitespace, the
expectation is to add markup around it, so for bold, we would add a `**` right
before the cursor and a `**` right after.

I will skip over the logic of identifying word boundaries and such, but that's
the bulk of the extra code in `toggleDecoration()`. Once we we know all the
ranges we need to toggle formatting for, and we know whether we need to toggle
it *on* or *off*, we simply call `markRange()` or `unmarkRange()` for each of
these ranges.

I used bold as an example throughout this post, but as you can see, the
functions are generic enough to apply to any type of inline formatting: bold,
italic, underline, etc. In fact, I have a bunch of functions wrapping these:

```typescript
export const toggleBold = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "strong", "**");

export const toggleItalic = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "em", "*");

export const toggleUnderline = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "underline", "~");

export const toggleStrikethrough = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "strikethrough", "~~");

export const toggleHighlight = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "highlight", "==");

export const toggleCode = (
    state: EditorState,
    dispatch: (tr: Transaction) => void
) => toggleDecoration(state, dispatch, "code", "`");
```

The app can invoke a command like *toggle bold* by calling the corresponding
`toggleBold()` function. Note everything is neatly wrapped in this function,
the caller just needs to pass in an editor state and a way to apply,
transactions, both being easy to get from an editor view.

Conversely, if we need to know whether the Bold button in a hypothetical toolbar
(at the time of writing, Flow doesn't have a toolbar) should render as pressed
or not, or whether the app menu or context menu should show a checkmark next to
the Bold menu item, we need to call `isRangeCovered()` and check whether the
range is fully bolded or not.

In a future post, I will focus on the commanding part, and how the editor makes
these functions available to the app.

## Block Formatting

There's more to formatting than what I covered in this post. Especially with
Markdown, but also generalizes quite well to other editors: there's *inline*
formatting which we just covered, and *block* formatting.

Inline formatting deals with marking up parts of text in a paragraph.

Block formatting deals with formatting a whole paragraph. Some Markdown examples
of this are:

* Headings - `#`, `##`, etc. at the beginning of a paragraph will make the
  whole paragraph be a heading.
* Block quotes - `>` at the beginning of a paragraph will make the whole
  paragraph be a block quote.
* Code blocks.
* Ordered and unordered lists - list handling brings a bunch more complexity,
  again a potential topic for its own blog post.

I will not cover block formatting since in most cases it is simpler than inline
formatting - the markers are always at the beginning of the paragraph and the
logic to determine whether toggling means toggling *on* or *off* is also
simpler.

List handling is in general more complex, especially if we need to support
things like nested lists and be smart about, for example, automatically
incrementing the numbering on a numbered list. That said, this has less to do
with formatting and more with how lists work.

## Summary

In this post, I covered some interesting aspects of how formatting works in an
editor (and what users would expect from a well-behaved editor):

* Toggling formatting *on* or *off* depending on the selection.
* Handling selection overlapping a mix of marked and unmarked text.
* Handling selection spanning multiple paragraphs by handling each paragraph
  separately.

I went over some of my implementation of this for Flow on top of ProseMirror
and introduced a few ProseMirror concepts like editor *state*, *decorations*,
and *transactions*.

Finally, I briefly touched on another, simpler flavor of formatting: *block*
formatting, which applies to whole paragraphs.

In the next post, I am planning to cover *commanding*, and how these lower-level
formatting functions can be packaged into a commanding system an app can present
to the user.

[^1]: See <https://prosemirror.net/docs/ref/#view.Decorations>.
[^2]: See <https://prosemirror.net/docs/ref/#state.Plugin_System>.
[^3]: See <https://prosemirror.net/docs/ref/#state.Transaction>.
