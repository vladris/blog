# Notes on Encoding Text

In this post we will look at text encoding, from the half-a-century old
ASCII and its extensions to Unicode and the common UTF-16 and UTF-8
encodings. We'll cover a few basic text encoding concepts like code
points, graphemes, and surrogate pairs and see how we can stack emojis
and combine them into intricate glyphs.

## ASCII

The American Standard Code for Information Exchange, or ASCII, was
developed in the 60s. The standard encodes each character in 7 bits, so
it has 128 characters. These include both the lowercase and uppercase
letters of the English alphabet, digits, punctuation, and a set of
control characters like LF (line feed, also known as `\n`), CR (carriage
return, also known as `\r`) or BEL (audible beep, or `\a`). The ASCII
character set contains all characters on a typewriter.

``` text
0x00 to 0x1F - control characters
0x20 to 0x2F - symbols - SPACE ! " # $ % & ' ( ) * + , - . /
0x30 to 0x39 - digits - 0 1 2 3 4 5 6 7 8 9
0x3A to 0x40 - symbols - : ; < = > ? @
0x41 to 0x5A - uppercase letters - A to Z
0x5B to 0x60 - symbols – [ \ ] ^ _ `
0x61 to 0x7A - lowercase letters - a to z
0x7B to 0x7E - symbols - { | } ~
0x7F - "delete" control character - DEL
```

### Parity bits

A byte is 8 bits wide, so the remaining bit was used as a *parity bit*.
The parity bit is 0 if the 7 bits encoding a character have an even
number of 1s and 1 if the bits encoding a character have an odd number
of 1s. For example, the letter `A` is encoded as `0x41`, or `1000001`.
`1000001` has an even number of bits so the parity bit is 0, thus `A`
ends up being encoded on 8 bits as `01000001`. The letter `C` is encoded
as `0x43`, or `1000011`, so the parity bit is 1. `C` ends up being
encoded on 8 bits as `11000011`.

The parity bit is used to check the consistency of the data: in a lossy
environment, a parity bit is a simple way to add extra validation. If
the parity bit does not have the expected value - for example it is 1
while the remaining 7 bits have an even number of bits set to 1 - it
means that the data is corrupted.

This encoding was enough for English, but lacked other letters common in
European languages, for example accented letters like é or ć. It also
could not encode any other alphabets, like Arabic, Cyrillic, Hebrew and
so on.

## Extended ASCII and Code Pages

The ASCII character set had numerous extensions which started to
leverage the 8th bit to encode information instead of using it to check
integrity. The original 128 character stayed in the `0x0` - `0x7F`
range, but with the extra bit, an extended ASCII character set could
encode 128 more characters from `0x80` to `0xFF`. Since ASCII received
multiple such extensions, these were distinguished by code pages. A code
page defined what the `0x80` to `0xFF` characters are.

IBM PC systems came with the popular **code page 437** which includes
characters for box-drawing like `║`, `╗`, and `╝`. Text-based user
interface could simulate windows, buttons and so on using these
characters:

``` text
╔════════════════════════════════════╗
║ Box-drawn window with progress bar ║
╟────────────────────────────────────╢
║  ██████████████████▓▒░░░░░░░░░░░░  ║
╚════════════════════════════════════╝
```

Another set of extensions was the **ISO/IEC 8859** standard consisting
of 16 parts, each adding support for additional character and alphabets.
For example, part 1 was named *Latin-1 Western European* and added most
characters required by Western European languages (like accented
letters), while part 5, *Latin/Cyrillic* used the available 128
characters to encode the Cyrillic alphabet.

While these code pages allowed new ranges of characters, each could only
add 128 symbols, which didn't scale well to the world's written
languages. For perspective, Kanji has thousands of characters.

## Unicode

The Unicode standard aims to cover most of the world's writing systems
and include additional symbols like emojis.

### Code points

Unicode introduces the notion of code points. Most code points are
characters, but some code points are used for formatting while some are
unassigned as of today and will be defined in future extensions of the
standard. The total code space of Unicode spans 1114112 code points,
from `0x0` to `0x10FFFF`.

The code space is divided into 17 planes, each consisting of a range of
65536 code points (from `0x0` to `0xFFFF`, from `0x10000` to `0x1FFFF`
and so on until `0xF0000` to `0x10FFFF`). A Unicode code point is
conventionally referred to as `U+XXX` where `XXX` is the hexadecimal
value of the code point. For example the code point corresponding to the
letter "a" which is `0x61`, is referred to as `U+0061`.

Most programming languages allow strings to contain a Unicode escape
sequence, which gets interpreted as a Unicode code point. Such sequences
start with `\x` or `\u` followed by the hexadecimal value of the code
point, for example in JavaScript the Unicode escape for "a" is the
string `"\u0061"`.

Not only does Unicode support a huge range of code points, it also
defines combining characters which, combined with other character,
create new characters. For example, ́ (`U+0301`) is the combining acute
accent. Combining this with "a" (`U+0061`) by appending it after the
letter (the string `"\u0061\u0301"`) results in the character "á"
while combining it with "e" (`U+0065`) in the string `"\u0065\u0301"`
results in the character "é".

Similarly, a skin tone modifier like dark skin tone, 🏿 (`U+1F3FF`) can
be appended to an emoji like the baby emoji 👶 (`U+1F476`) to get a baby
with a dark skin tone 👶🏿.

### Graphemes

A grapheme is a graphical symbol that a user sees on the screen. Text
rendering is done through graphemes. Some graphemes correspond to a
single code point, like "a" corresponding to `U+0061`. Other graphemes
correspond to a sequence of code points, like 👶🏿, which corresponds to
`U+1F476` then `U+1F3FF`. There are also graphemes which can be obtained
in multiple ways. The accented "e" in the example above, "é", can be
obtained by combining the letter "e" with the acute accent (`U+0065`
then `U+0301`), but there is also an accented "e" character "é"
represented by the code point `U+00E9`. Both `U+0065 U+0301` and
`U+00E9` resolve to the same grapheme.

Because of such equivalences, the standard defines a normalization
procedure which can convert equivalent texts to the same code point
representation. There are several ways to achieve this, which we'll not
cover in this blog post.

Combining and modifying characters can be stacked one after the other.
For example, the 👨‍❤️‍👨 emoji showing two man with a heart above is a single
grapheme but consists of the following sequence of code points:
`U+1F468 U+200D U+2764 U+FE0F U+200D U+1F468`. This is a combination of:

* The man emoji 👨 (`U+1F468`)
* The zero-width joiner (`U+200D`) which does not have a stand-alone
  representation but combines two emojis into a single one
* The heavy black heart symbol ❤ (`U+2764`) - depending on which OS
  you are reading this, it might or might not render as an emoji
* The Variation Selector-16 character (`U+2764`) which also doesn't
  have a stand-alone representation but can be applied to code points
  which have both a text and an emoji representation to select the
  emoji representation. This ensures the heavy black heart symbol gets
  the emoji representation ❤️.
* Another zero-width joiner (`U+200D`)
* Another man emoji 👨 (`U+1F468`)

The whole sequence results in a single grapheme.

One interesting thing to note is that splitting a string without being
aware of how the code points combine can change the representation of
the text. This can happen when breaking a line of text to fit on screen.
In the above example, even though we have 6 code points, we end up with
a single grapheme, so when dealing with rendering, it's usually best to
operate on graphemes not code points/characters.

## Encodings

We talked about code points and graphemes, but how are the code points
actually encoded as bytes? With ASCII and the simple extensions, the
encoding was easy, as each byte encoded a character. Unicode has over a
million code points, so let's look at how these get translated into
bytes.

### UTF-32

The most obvious way is to find the minimum number of bytes that can
encode any code point. Since code points range from `0x0` to `0x10FFFF`,
we need 21 bits to store all possible values (`0x10FFFF` in binary is
`0b100001111111111111111`).

Because most CPUs nowadays have a word size of at least 32-bits, the
UTF-32 encoding rounds up the number of required bits from 21 to 32,
thus representing a code point using 4 bytes.

This encoding is very straight-forward, as any 4 bytes store the value
of a code point in a string, but it is also very space-inefficient. The
leading bits are always 0 and not only that, the code points
representing common characters and alphabets appear in the lower planes,
so while an emoji like the woman emoji 👩 (`U+1F469`) in binary is
`0b11111010001101001`, thus requiring at least 17 bits to represent, the
code point for the letter "a" is `U+0061`, same as the old ASCII
representation. That is `0b1100001` in binary, requiring only 7 bits.

To take advantage of this, several *variable-length* encodings were
developed, which use fewer bytes for code points representable with a
smaller number of bits, and more bytes for higher code point values. The
two most common encodings are UTF-16 and UTF-8.

### UTF-16

UTF-16 encodes code points in either one or two 16-bit wide code units.
The code points in the range `U+0000` to `U+FFFF` are encoded directly
as a 16-bit code unit, except the subrange `U+D800` to `U+DFFF` which
we'll talk about shortly. This range corresponds to Plane 0, the *Basic
Multilingual Plane*, with code points to represent almost all modern
languages.

Code points from other planes are encoded in UTF-16 using two code
units, so 32 bits. Any code point in the range `U+10000` to `U+10FFFF`
(or `0x10000` to `0x10FFFF`) is encoded by subtracting `0x10000` leaving
a value between `0x0` and `0xFFFFF`. Values in this range can be
represented in 20 bits. A sequence of 10 bits can represent values in
the range `0x0` to `0x3FF`. The 20 bits are split into the first (most
significant) 10 bits and the last (least significant) 10 bits. The first
10 bits are added to `0xD800`, resulting in a value in the range
`0xD800` to `0xDBFF`. This is represented in the first 16-bit code
point. The last 10 bits are added to `0xDC00`, resulting in a value in
the range `0xDC00` to `0xDFFF`. This is represented in the second 16-bit
code point.

Let's take as an example the woman emoji 👩 (`U+1F469`). The UTF-16
encoding goes as follows:

* Subtract `0x10000` from `0x1F469`, resulting in `0xF469`, or
  `0b00001111010001101001` in 20 bits.
* The first 10 bits, `0b0000111101`, or `0x3D`, are added to `0xD800`
  which gives us `0xD83D`.
* The remaining 10 bits, `0b0001101001`, or `0x69`, are added to
  `0xDC00`, giving `0xDC69`.

The two 16-bit code units for 👩 are `0xD83D` and `0xDC69`, or the byte
sequence `0xD8 0x3D 0xDC 0x69`.

**Surrogate pairs**

We said that UTF-16 encodes all code points in Plane 0 using a single
16-bit code unit, except the range `U+D800` to `U+DFFF`. That particular
range is reserved in the Unicode standard for UTF-16 surrogate pairs, so
code points in that range are unassigned and will never be assigned.

If we review the way UTF-16 encodes code points in code units, a 16-bit
code unit can be either:

* A value in the range `0x0` to `0xFFFF`, except the reserved range
  `0xD800` to `0xDFFF`. This value is a valid code point in Plane 0.
* A value in the range `0xD800` to `0xDBFF`, which represents the
  first 10 bits of a code point in another plane after subtracting
  `0x10000` and adding `0xD800` to the first 10 bits
* A value in the range `0xDC00` to `0xDFFF`, which represents the
  remaining 10 bits of a code point in another plane after subtracting
  `0x10000` and adding `0xDC00` to the last 10 bits.

Note that these ranges are disjoint - a value can appear in only one of
them, so each 16-bit code unit can unambiguously be identified, in
isolation. For code points like 👩, encoded as two 16-bit code units,
the code units are called a *surrogate pair*. The first code unit, in
the range `0xD800` to `0xDBFF` is called the *high surrogate* while the
second code unit, in the range `0xDC00` to `0xDFFF` is called the *low
surrogate*.

Since the Unicode standard and UTF-16 encoding evolved together, the
range `0xD800` to `0xDFFF` needed by the surrogate pairs was reserved in
Plane 0 and the code points were kept unassigned. Without this, UTF-16
would have had trouble encoding a code point in that range as it would
become undistinguishable from a surrogate.

UTF-16 is the default encoding used by Windows, Java, the .NET runtime,
and JavaScript. Another popular way to encode text is UTF-8.

## UTF-8

UTF-8 uses 8-bit code units, so it encodes code points using one to four
bytes. To recap, Unicode code points can be represented in 21 bits, as
their valid range is between `0x0` and `0x10FFFF`.

UTF-8 encodes code points as follows:

* Code points in the range `U+0000` to `U+007F` are represented with a
  single byte with the 8th (most significant) bit being 0:
  `0b0xxxxxxx`.
* Code points in the range `U+0080` to `U+07FF` are represented with
  two bytes. The first byte starts with the bits `0b110xxxxx`, the
  second byte is `0x10xxxxxx`. Without the prefixes, there are 11 bits
  used to encode the code point (count the number of `x`s ).
* Code points in the range `U+0800` to `U+FFFF` are represented with
  three bytes. The first byte starts with the prefix `0b1110xxxx`, the
  following two bytes with `0b10xxxxxx`. This leaves 16 bits to encode
  the code point.
* Code points in the range `U+10000` to `U+10FFFF` are represented
  with four bytes. The first byte starts with the prefix `0b11110xxx`,
  the following three bytes with `0b10xxxxxx`. This leaves 21 bits to
  encode the code point.

This encoding has several interesting properties: for code points in
lower planes, it is more compact than UTF-16. UTF-16 requires either one
or two 16-bit code units, while UTF-8 can use 8, 16, 24, or 32 bits
depending on the code point. Commonly used alphabets are in the lower
planes, so usually fewer bits are needed.

UTF-8 is also ASCII-compatible: the first 128 characters (`U+0000` to
`U+007F`), represented in 7 bits, are the same as the old ASCII
encoding. An ASCII string can be used directly as UTF-8 encoded text
without any transformations required.

Unlike UTF-16, which can uniquely distinguish each code unit as either a
code point, a high surrogate, or a low surrogate, with UTF-8 we cannot
always determine what a code unit is in isolation: `0b10110011` could be
the second, third, or fourth byte in a code point. This is a consequence
of the more compact encoding. On the other hand, with UTF-8 we can look
at the bit prefix and determine the length of the sequence:

* If the prefix of the byte is `0b0xxxxxxx`, we have an ASCII
  character.
* If the prefix is `0b110xxxxx`, we are looking at the first byte of a
  code point encoded in 2 bytes.
* If the prefix is `0b1110xxxx`, we are expecting a sequence of 3
  bytes.
* If the prefix is `0b11110xxx`, we are expecting a sequence of 4
  bytes.
* If the prefix is `0b10xxxxxx`, we know we aren't looking at the
  first byte in a sequence, rather at a byte inside a sequence.

Note the bit patterns do not overlap. Beyond ASCII, the number of 1 bits
in the prefix coincides with the number of bytes used to encode the code
point.

As an example, let's take the same 👩 emoji and see how its encoding
looks like in UTF-8. 👩 is the code point `U+1F469`, so it requires 4
bytes. `0x1F469` represented in binary with 21 bits is
`0b000011111010001101001`.

We fill this into `0b11110xxx 0b10xxxxxx 0b10xxxxxx 0b10xxxxxx`, which
gives us `0b11110000 0b10011111 0b10010001 0b10101001`. In hexadecimal,
this is `0xF0 0x9F 0x91 0xA9`. This is the encoding of our emoji in
UTF-8.

UTF-8 is the default encoding used by Linux and macOS. It is also the
standard for the internet, with a majority of web pages using this
encoding.

Another important thing to keep in mind when manipulating text is how it
is encoded. When reading a sequence of bytes from a file or a network
connection, we need to make sure we don't mistakenly try to interpret
UTF-8 encoded text as UTF-16 encoded text or vice-versa. Since different
systems default to different encodings, this is a very plausible
scenario.

## Summary

In this post we looked at some common text encoding standards and
concepts:

* ASCII, which encodes 128 characters.
* Extended ASCII, which encodes 256 characters:
  * Code page 437, with box-drawing characters
  * The ISO/IEC 8859 16-part standard with code pages for various
    alphabets
* Unicode:
  * Code points and planes
  * Graphemes and combining characters
* Encodings:
  * The inefficient UTF-32
  * UTF-16 and surrogate pairs
  * Popular UTF-8
