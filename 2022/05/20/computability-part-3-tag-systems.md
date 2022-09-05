# Computability Part 3: Tag Systems

In the [previous
post](https://vladris.com/blog/2022/04/03/computability-part-2-turing-machines.html)
we talked about universal Turing machines and looked at some very small
machines that are still capable of computing anything that can be
computed (the Turing-completeness property). In this post, we'll look
at another model for computation: tag systems.

> A tag system operates on a string of symbols by reading the symbol
> from the head of the string, deleting a constant number of symbols
> from the head of the string, and appending one or more symbols to the
> tail of the string based on the symbol read from the head.

Formally:

> A tag system is a triplet $\langle m, A, P \rangle$.
>
> * $m$ is a positive integer, called the *deletion number*, which
>   specifies how many symbols are deleted from the head during each
>   iteration.
> * $A$ is a finite alphabet of symbols, including a special *halting
>   symbol*.
> * $P$ is a set of *production rules* which map each symbol in $A$ to
>   a string of symbols or *words* from $A$ (to be appended to the end
>   of the string).

Tag systems were specified by Emil Leon Post in 1943, 7 years after
Turing Machines. We usually refer to tag systems as *m-tag systems*
where $m$ is the deletion number from the definition above.

At each step, $x$ is read from the head of the string, $m$ symbols are
deleted, and $P(x)$ is appended to the end of the string. The tag system
halts when $x$ is the halting symbol.

An alternative definition that doesn't require a halting symbol
considers as halting all words that are smaller than $m$. In this case,
the tag system halts when the string shrinks sufficiently. Yet another
alternative considers as halting the empty string. In this case, the tag
system halts when the string becomes empty.

Let's look at a Python implementation for a tag system:

``` python
def tag_system(m, productions, string):
    # Repeat until the string is empty or we see the halting symbol
    while string and string[0] in productions:
        string = string[m:] + productions[string[0]]

        yield string
```

As an example, let's take the tag system with
$m = 2, A = \langle a, b, H \rangle$, and the production rules

|Symbol | Word |
|-------|------|
| a     | aab  |
| b     | H    |

Starting with the string `aa`, the steps are:

``` text
aa              // Erase 2 symbols from head, a -> aab
  aab           // Erase 2 symbols from head, a -> aab
    baab        // Erase 2 symbols from head, b -> H
      abH       // Erase 2 symbols from head, a -> aab
        Haab    // Halt
```

Using our `tag_system()` function implemented above:

``` python
productions = {
    'a': 'aab',
    'b': 'H',
}

string = 'aa'

print(string)
for string in tag_system(2, productions, string):
    print(string)
```

Tag systems are simple, even simpler than Turing machines. Remember we
defined a Turing machine as a 7-tuple while tag systems are represented
by triplets. Turing machines have states, and depending on the state, a
machine takes different actions. Tag systems technically have a single
state: when a symbol is read from the head of the string, the same thing
will always happen: $m$ symbols are deleted from the head and the
corresponding production rule determines what word to append to the tail
of the string. Even so, tag systems are Turing-complete.

## Turing completeness

For $m \gt 1$, m-tag systems are Turing complete. For any Turing
machine, there is an m-tag system that can emulate that Turing machine.
John Cocke and Marvin Minsky showed in 1964 how a 2-tag system can
emulate a universal Turing machine[^1]. That means that such a super
simple system is also capable of universal computation!

But it gets even simpler.

## Cyclic tag systems

A cyclic tag system is a modification of tag systems where:

* $m = 1$: only one symbol is deleted from the head of the string.
* The alphabet consists of only `0` and `1`.
* Instead of production rules, we use a finite list of words (on the
  alphabet consisting of only `0` and `1`) called *productions*.

Instead of production rules, we cycle through the list of productions.
We start from the head of the list of productions. At each step, if the
symbol at the head of the string is `1`, we append the production to the
end of the string. If the symbol at the head of the string is `0`, we
don't append anything. We then move to the next production in the list
for the next step. Once we exhaust the list of productions, we loop
around to the head (this inspired the *cyclic* name).

Here is a Python implementation for a cyclic tag system:

``` python
def cyclic_tag_system(productions, string):
    # Keeps track of current production
    i = 0

    # Repeat until the string is empty
    while string:
        string = string[1:] + (productions[i] if string[0] == '1' else '')

        # Update current production
        i = i + 1
        if i == len(productions):
            i = 0

        yield string
```

For example, we will use the production rules `11`, `01`, `00`. With an
initial string `1`, the steps are:

``` text
1               // Append production 11
 11             // Append production 01
  101           // Append production 00
   0100         // Current production 11 (won't append since head is 0)
    100         // Append production 01
     0001       // Current production 00 (won't append since head is 0)
      001       // Current production 11 (won't append since head is 0)
       01       // Current production 01 (won't append since head is 0)
        1       // Append production 00
         00     // Current production 11 (won't append since head is 0)
          0     // Current production 01 (won't append since head is 0)
                // Halts
```

Using our Python implementation:

``` python
productions = ['11', '01', '00']

string = '1'

print(string)
for string in cyclic_tag_system(productions, string):
    print(string)
```

Cyclic tag systems are simpler than tag systems since $m$ is fixed to
`1`, the alphabet is fixed to `0` and `1`, and productions are a
represented as a cyclic list rather than a map of symbols to words. Even
so, a cyclic tag system can emulate any m-tag system.

## Emulating tag systems with cyclic tag systems

An m-tag system with $n$ symbols $\lbrace a_1, a_2, ... a_n \rbrace$ and
their corresponding production rules $\lbrace P_1, P_2, ... P_n \rbrace$
can be translated to a cyclic tag system with $m * n$ productions where
the first $n$ productions $\lbrace P'_1, P'_2, ... P'_n \rbrace$ are
encodings of their respective $P$-productions in the m-tag system and
the rest are empty strings.

Productions in the m-tag system are words over the alphabet $A$. We
encode each symobl in $A$ as a binary string of length $n$, with a `1`
in the $k$-th position for $a_k$. For example, for $n = 3$ and the
alphabet $A = \lbrace a_1, a_2, a_3 \rbrace$, we encode $a_1$ as `100`,
$a_2$ as `010`, $a_3$ as `001`. Since a production $P_k$ is a sequence
of symbols, we can similarly translate it into an encoded representation
$P'_k$ using symbols `0` and `1`.

Our first example was the 2-tag system with the alphabet
$A = \langle a, b, H \rangle$, and the production rules

|Symbol | Word |
|-------|------|
| a     | aab  |
| b     | H    |
| H     | H    |


Here we added the production rule `H -> H` for completeness, so we have
exactly $n$ production rules.

Translating this into a cyclic tag system, $a, b, H$ become `100`,
`010`, and `001` respectively. The production rules translate as:

> `a -> aab` becomes `100100010`
>
> `b -> H` becomes `001`
>
> `H -> H` becomes `001`

The full list of production for the cyclic tag system is
`100100010, 001, 001, -, -, -` where `-` is the empty string.

The initial string `aa` becomes `100100`, so our emulation is:

``` text
100100                      // * Emulated production rule a -> aab
 00100100100010             // P = 001 (but head is 0)
  0100100100010             // P = 001 (but head is 0)
   100100100010             // P = empty string
    00100100010             // P = empty string, head is 0
     0100100010             // P = empty string, head is 0
      100100010             // * Emulated production rule a -> aab
       00100010100100010    // P = 001 (but head is 0)
        0100010100100010    // P = 001 (but head is 0)
         100010100100010    // P = empty string
          00010100100010    // P = empty string, head is 0
           0010100100010    // P = empty string, head is 0
            010100100010    // P = 100100010 (but is 0)
             10100100010    // * Emulated production rule b -> H
              0100100010001 // P = 001 (but head is 0)
               100100010001 // P = empty string
                ...
```

Using our Python implementation:

``` python
productions = ['100100010', '001', '001', '', '', '']

string = '100100'

print(string)
for string in cyclic_tag_system(productions, string):
    print(string)
```

Note in this case the cyclic tag system won't halt when the emulated
m-tag system halts, since that would be an emulated halt. But we can
stop it by checking whether the first 3 symbols represent the encoding
of `H`. We do this every sixth step, since we have a 2-tag system with 3
symbols, which means we emulate 1 step of the tag system with 6 steps of
the cyclic tag system.

``` python
productions = ['100100010', '001', '001', '', '', '']

i, string = 0, '100100'

print(string)
for string in cyclic_tag_system(productions, string):
    print(string)

    i = (i + 1) % 6

    # Break if halting symbol is at the head of the string
    if i == 0 and string[:3] == '001':
        break
```

Or, an updated example that prints every sixth step and translates from
the cyclic tag system encoding to the original symbols:

``` python
productions = ['100100010', '001', '001', '', '', '']

symbols = {
    '100': 'a',
    '010': 'b',
    '001': 'H',
}

def translate(s):
    return ''.join([symbols[s[i:i + 3]] for i in range(0, len(s), 3)])

i, string = 0, '100100'

print(f'{string} ({translate(string)})')
for string in cyclic_tag_system(productions, string):
    i = (i + 1) % 6
    if i == 0:
        print(f'{string} ({translate(string)})')
        if string[:3] == '001':
            break
```

Running this code should be the emulated equivalent of our first example
in this post.

Since m-tag systems (with $m \gt 1$) are Turing-complete and cyclic tag
systems can emulate any m-tag system, it follows that cyclic tag systems
are also Turing complete. We can compute anything that is computable
with the alphabet `0`, `1`, and a list of words over this alphabet!

In the next post, we will continue our exploration of simple systems
capable of universal computation with cellular automata.

[^1]: See [Universality of Tag Systems With P =
    2](https://dl.acm.org/doi/pdf/10.1145/321203.321206).
