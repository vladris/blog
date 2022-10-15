# Computability Part 8: Lambda Calculus

In the previous posts, we dug deeper into one particular model of computation,
starting with Turing Machines in [part 2](https://vladris.com/blog/2022/04/03/computability-part-2-turing-machines.html),
to the von Neumann computer architecture in [part 6](https://vladris.com/blog/2022/07/31/computability-part-6-von-neumann-architecture.html),
to some of the implementation practicalities of machines - physical or virtual
- in [part 7](https://vladris.com/blog/2022/09/02/computability-part-7-machine-implementation-practicalities.html).

We'll switch gears and cover another computational model this time around:
*lambda calculus*. Lambda calculus was developed by Alonzo Church around the
same time Alan Turing was proposing the Turing machine as a universal model for
computation. The *Church-Turing thesis*[^1] proves the equivalence between the
two models - anything a Turing machine can compute can also be computed by
lambda calculus.

Formally:

> Lambda calculus consists of lambda terms and reductions applied to lambda
> terms.
> 
> The lambda terms are built with the following rules, where $\Lambda$ is the
> set of all possible lambda terms:
>
> * *Variables*, like $x$, are lambda terms. $x \in \Lambda$.
> * *Abstractions*, $(\lambda x.M)$. This is a function definition where $M$ is
>   a lambda term and $x$ becomes *bound* in the expression. For $x \in \Lambda$
>   and $M \in \Lambda$, $(\lambda x.M) \in \Lambda$. 
> * *Applications*, $(M \space N)$. This applies the function $M$ to the
>   argument $N$, where $M$ and $N$ are lambda terms. For $M \in \Lambda$ and $N
>   \in \Lambda$, $(M \space N) \in \Lambda$.
>
> If a term $y$ appears in $M$ but is not bound, then $y$ is *free* in $M$, e.g.
> for $\lambda x.y \space x$, $x$ is bound and $y$ is free.
> The reductions are:
>
> * *$\alpha$-equivalence*: bound variables in an expression can be renamed to
>   avoid collisions: $(\lambda x.M[x]) \rightarrow (\lambda y.M[y])$.
> * *$\beta$-reduction*: bound variables in the body of an abstraction are
>   replaced with the argument expression: $(\lambda x.t)s \rightarrow
    t[x := s]$.
> * *$\eta$-reduction*: if $x$ is a variable that does not appear free in the
>   lambda term *M*, then $\lambda x.(M x) \rightarrow M$. This can also be
>   understood in terms of function equivalence: if two functions give the same
>   result for all arguments, then the functions are equivalent.

Let's look at a few simple examples in Python:

``` python
lambda x: x
```

This is the identity function expressed as a lambda abstraction. In this case,
`x` (the lambda parameter), becomes bound in the body of the lambda.

$\alpha$-equivalence:

``` python
lambda y: y
```

This is the same identity function, we're just using `y` instead of `x` to name
the parameter.

For function application, we can apply the identity function to any other lambda
term and get back that lambda term:

``` python
(lambda x: x)(lambda y: y)
```

This applied the identify function `lambda x: x` to the argument `lambda y: y`,
which will give us back `lambda y: y`.

## Church encoding

Based on the above definition, lambda calculus consists *exclusively* of lambda
terms - while `(lambda x: x)(10)` is valid Python code, applying an identity
lambda to the number `10`, lambda calculus does not have a number `10`. Enter
*Church encoding*: Alonzo Church came up with a way to encode logic values and
numbers as lambda terms.

### Logic

Let's start with Boolean logic: `TRUE` is defined as $T := (\lambda x.\lambda
y.x)$, `FALSE` is defined as $F := (\lambda x.\lambda y.y)$.

``` python
TRUE = lambda x: lambda y: x
FALSE = lambda x: lambda y: y
```

Note with this definition, if we apply a first argument to `TRUE`, and a second
argument to the returned lambda, we always get back the first argument. For
`FALSE`, we always get back the second argument.

We can defined `IF` as $IF := (\lambda x.x)$. This is the same as the identity
function.

``` python
IF = lambda x: x
```

This works since we defined `TRUE` to always return the first argument and
`FALSE` to always return the second argument. So when we call `IF(c)(x)(y)`,
if `c` is `TRUE`, we get back `x` (the if-branch), otherwise we get back `y`
(the else-branch).

We can try this out (though again this is outside of lambda calculus, we are
introducing numbers for clarity):

``` python
IF(TRUE)(1)(2)  # This returns 1
IF(FALSE)(1)(2) # This returns 2
```

Now that we can express if-then-else, we can easily express other logic
operators. Negation is $\lambda x.(x \space F \space T)$.

``` python
NOT = lambda x: x(FALSE)(TRUE)
```

If `x` is `TRUE`, we get back the first argument, `FALSE`; if `x` is `FALSE`,
we get back the second argument, `TRUE`.

`x AND y` can be expressed as *if x then y else FALSE*, or: $\lambda x.\lambda
y.(x \space y \space F)$. `x OR y` can be expressed as *if x then TRUE else y*,
or $\lambda x.\lambda y.(x \space T \space y)$.

``` python
AND = lambda x: lambda y: x(y)(FALSE)
OR = lambda x: lambda y: x(TRUE)(y)
```

Here are a few examples:

``` python
print(AND(TRUE)(TRUE) == TRUE)  # prints True
print(AND(TRUE)(FALSE) == TRUE) # prints False
print(OR(TRUE)(FALSE) == TRUE)  # prints True
print(NOT(FALSE) == TRUE)       # prints True
```

Using only lambda terms, we were able to implement Boolean logic! But Church
encoding goes further - we can also represent natural numbers and arithmetic
as lambda terms.

### Arithmetic

Alonzo Church encoded numbers as applications of a function $f$ to a term $x$.

* `0` means applying $f$ 0 times to the term: $0 := \lambda f.\lambda x.x$.
* `1` means applying $f$ once to the term: $1 := \lambda f.\lambda x.f x$.
* `2` means applying $f$ twice: $2 := \lambda f.\lambda x.f (f x)$.

In general, the number `n` is represented by `n` applications of `f`: $n :=
\lambda f.\lambda x.f (f (... (f x)) ... ))$ or $n := \lambda f.\lambda x.
f^n(x)$.

In Python:

``` python
ZERO = lambda f: lambda x: x
ONE = lambda f: lambda x: f(x)
TWO = lambda f: lambda x: f(f(x))
...
```

Note `ZERO` is the same as `FALSE`. With this definition of numbers, we can
define the successor function `SUCC` as a function that takes a number `n`
(represented with our Church encoding), the function `f`, the term `x`, and
applies `f` one more time. $SUCC := \lambda n.\lambda f.\lambda x.f (n f x)$.

``` python
SUCC = lambda n: lambda f: lambda x: f(n(f)(x))
```

We can define addition as $PLUS := \lambda m.\lambda n.m \space SUCC \space n$.
Since we define a number as repeatedly applying a function, we express `m + n`
as applying `m` times the successor function `SUCC` to `n`.

``` python
PLUS = lambda m: lambda n: m(SUCC)(n)
```

We can similarly define multiplication as applications of the `PLUS` function:

``` python
MUL = lambda m: lambda n: m(PLUS)(n)
```

We'll stop here with arithmetic, but this should hopefully give you a sense of
the expressive power of lambda calculus.

## Combinators

Some well-known lambda terms are called *combinators*:

* $I$ is the identity combinator $I := \lambda x.x$.
* $K$ is the constant combinator $K := \lambda x.\lambda y.x$. When applied to
  an argument $x$, it returns a constant function $K_x$ which returns $x$ when
  applied to any argument.
* $S$ is the substitution combinator $S := \lambda x.\lambda y.\lambda z.x z
  (y z)$. $S$ takes 3 arguments, $x$, $y$, and $z$, applies $x$ to $z$, then
  applies the result of applying $y$ to $z$ to it.

In Python:

``` python
I = lambda x: x
K = lambda x: lambda y: x
S = lambda x: lambda y: lambda z: x(z)(y(z))
```

Turns out these 3 combinators can together express any lambda term. The SKI
combinators are the simplest "programming language" since they can express
anything expressable in lambda calculus, which we know is Turing-complete.

### The Y combinator

Another interesting combinator is the $Y$ combinator. In lambda calculus, there
is no way for a function to reference itself: within the body of a lambda like
`lambda x: ...` we can refer to the bound term `x`, but we can reference the
lambda itself. The implication is that we can't define, using this syntax,
self-referential functions. We can only pass functions as arguments. How can we
then implement recursion? With the $Y$ combinator, of course.

Let's take an example: we can recursively define factorial as:

``` python
def fact(n):
    return 1 if n == 0 else n * fact(n - 1)
```

This works, but note we reference `fact()` within its body. In lambda calculus
we can't do that.

The $Y$ combinator is defined as $Y := \lambda f.(\lambda x.f (x x))(\lambda
x.f (x x))$.

``` python
Y = lambda f: (lambda x: f(x(x)))(lambda x: f(lambda z: x(x)(z)))
```

Note the Python implementation is slightly different than the mathematical
definition. This has to do with the way in which Python evaluates arguments.
We won't go into the details here, but consider this a Python implementation
detail irrelevant to the lambda calculus discussion[^2].

Here is a lambda version of factorial:

``` python
FACT = lambda f: lambda n: 1 if n == 0 else n * f(n - 1)
```

With this definition, we pass the function to call as an argument (`f`). We can
fully express this in lambda calculus (using Church numerals, arithmetic and
logic), but we'll keep the example simple. We can then use the $Y$ combinator
like this:

``` python
print(Y(FACT)(5))  # prints 120
```

This should give you an intuitive understanding of how the $Y$ combinator
works: we pass it our function and argument, and it enables the recursion
mechanism.

We can similarly implement Fibonacci as:

``` python
FIB = lambda f: lambda n: 1 if n <= 2 else f(n - 1) + f(n - 2)

print(Y(FIB)(10))  # prints 55
```

The powerful $Y$ combinator can be used to define recursive functions in
programming languages that don't natively support recursion.

# Lists

Let's also look at how we can express lists in lambda calculus. Let's start
with pairs. We can define a pair as $PAIR := \lambda x.\lambda y.\lambda f.
f x y$. We can extract the first element of a pair with $FIRST := \lambda p.
p \space T$ and the second one with $SECOND := \lambda p.p \space F$.

``` python
PAIR = lambda x: lambda y: lambda f: f(x)(y)
FIRST = lambda p: p(TRUE)
SECOND = lambda p: p(FALSE)

print(FIRST(PAIR(10)(20)))  # prints 10
print(SECOND(PAIR(10)(20))) # prints 20
```

We can define a `NULL` value as $NULL := \lambda x.T$ and a test for `NULL` as
$ISNULL := \lambda p.p (\lambda x.\lambda y.FALSE)$.

``` python
NULL = lambda x: TRUE
ISNULL = lambda p: p(lambda x: lambda y: FALSE)
```

We can now define a linked list as either `NULL` (an empty list) or as a pair
consisting of a pair of elements - a head element and a tail list.

We can get the head of the list using `FIRST` and the tail using `SECOND`. Given
list $L$, we can prepend an element $x$ by forming the pair $(x, L)$.

``` python
HEAD = FIRST
TAIL = SECOND
PREPEND = lambda x: lambda xs: PAIR(x)(xs)
```

We can build a list by prepending elements to `NULL`, and traverse it using
`HEAD` and `TAIL`:

``` python
# Build the list [10, 20, 30]
L = PREPEND(10)(PREPEND(20)(PREPEND(30)(NULL)))

print(HEAD(TAIL(L))) # prints 20
```

Appending is more interesting: if our list is represented as a pair of head and
tail, we need to "traverse" the list until we reach the end. This sounds a lot
like a recursive function: appending `x` to `xs` entails returning the pair
`PAIR(x, NULL)` if `xs` is `NULL`, else the pair `PAIR(HEAD(xs), APPEND(TAIL(xs,
x)))`. Fortunately, we just looked at the $Y$ combinator which allows us
to express this.

Here is a simplified, readable implementation, using Python tuples:

``` python
_append = lambda f: lambda xs: lambda x: \
    (x, None) if not xs else (xs[0], f(xs[1])(x))

append = Y(_append)

print(append(append(append(None)(10))(20))(30))

# This will print (10, (20, (30, None)))
```

We can express the same using the lambdas we defined above (`NULL`, `ISNULL`,
`PAIR`, `HEAD`, `TAIL`):

``` python
_APPEND = lambda f: lambda xs: lambda x: \
    ISNULL(xs) (lambda _: PAIR(x)(NULL)) (lambda _: PAIR(HEAD(xs))(f(TAIL(xs))(x))) (TRUE)

APPEND = Y(_APPEND)

L = APPEND(APPEND(APPEND(NULL)(10))(20))(30)

print(HEAD(L))       # prints 10
print(HEAD(TAIL(L))) # prints 20
```

We covered logic, arithmetic, combinators, pairs, and lists, all expressed as
lambda terms. Let's also sketch a proof of Turing completeness, like we did in
previous posts.

## A sketch of Turing completeness

We're calling this a "sketch", as lambda notation is not easy to read. We will
instead look at an implementation using more Python syntax than just lambdas,
but we will only use constructs which we know can be expressed in lambda
calculus.

As usual, we will emulate another system which we know to be Turing-complete.
In [part 3](https://vladris.com/blog/2022/05/20/computability-part-3-tag-systems.html)
we looked at tag systems. We talked about cyclic tag systems, which can emulate
m-tag systems, which are Turing-complete. As a reminder, a cyclic tag system is
implemented as a set of binary strings (strings containing only `0`s and `1`s)
which are production rules, and we process a binary input string by popping the
head of the string and, if it is equal to `1`, appending the current production
rule to the string. We cycle through the production rules at each step. This is
the code we used in the previous post:

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

We used the productions `11`, `01`, and `00` and the input `1`:

``` python
productions = ['11', '01', '00']

string = '1'

print(string)
for string in cyclic_tag_system(productions, string):
    print(string)
```

Let's sketch an alternative implementation using the constructs we covered in
this post.

First, we can describe our production rules as lists of Boolean values. We
know how to represent Boolean values (`TRUE` and `FALSE`), and how to build
a list using `PAIR`. Our productions can be represented as:

``` python
p1 = (True, (True, None))   # PAIR(TRUE)(PAIR(TRUE)(NULL))
p2 = (False, (True, None))  # PAIR(FALSE)(PAIR(TRUE)(NULL))
p3 = (False, (False, None)) # PAIR(FALSE)(PAIR(FALSE)(NULL))

productions = (p1, (p2, (p3, None)))
```

We can cycle through the list by processing the head, then appending it to
the tail of the list. Here are simpler implementations of our list processing
functions over Python tuples (though we know how to do these using only lambda
terms):

``` python
def head(p):
    return p[0]

def tail(p):
    return p[1]

def append(xs, x):
    return (x, None) if not xs else (head(xs), append(tail(xs), x))

# If we want to cycle through our productions, we can do:
# productions = append(tail(productions), head(productions))
```

We'll also need a function to concatenate two lists. We can easily build this
on top of `append()`:

``` python
def concat(xs, ys):
    return xs if not ys else concat(append(xs, head(ys)), tail(ys))
```

While we still have `ys`, we append the head of `ys` to `xs`, then recurse
with the tail of `ys`.

We process our input string as follows: if it is empty, we are done. If not,
if the head is `1`, we concatenate our current production to the end of the
string, and recurse, cycling productions:

``` python
def cyclic_tag_system(productions, input):
    return None if not input else \
        cyclic_tag_system(
            # Cycle productions
            append(tail(productions), head(productions)),
            # If head is True, concatenate head production. Pop head input either way.
            concat(tail(input), head(productions)) if head(input) else tail(input))
```

Let's throw in a `print()` and run this on the same input as our original
example:

``` python
def cyclic_tag_system(productions, input):
    print(input)
    return None if not input else \
        cyclic_tag_system(
            # Cycle productions
            append(tail(productions), head(productions)),
            # If head is True, concatenate head production. Pop head input either way.
            concat(tail(input), head(productions)) if head(input) else tail(input))

# The input is equivalent to the string '1'
cyclic_tag_system(productions, (True, None))
```

This should produce output very similar to our original `cyclic_tag_system()`,
but using lists of Booleans instead of strings of `0`s and `1`s.

We emulated a cyclic tag system in lambda calculus - well, we didn't write all
the code as lambda terms, but everything is expressed as one-liner functions
that use only if-then-else expressions, lists (pair, head, tail), and recursion
(for which we have the $Y$ combinator).

Lambda calculus has been extremely influential in computer science - it is the
root of functional programming. LISP, one of the earliest programming
languages, is heavily influenced by lambda calculus. Many ideas, like anonymous
functions, also known as *lambdas*, are now broadly available in most modern
programming languages (Python even uses the keyword `lambda` for these, as we
saw in this post).

## Summary

In this post we covered lambda calculus:

* Lambda terms, including *variables*, *abstractions*, and *applications*.
* Reductions: $\alpha$-equivalence, $\beta$-reduction, and $\eta$-reduction.
* Church encoding for Boolean logic and arithmetic using lambda terms.
* Combinators: the $S$, $K$, and $I$ combinators which are sufficient to encode
  all lambda terms, and the $Y$ combinator which enables recursion.
* Pairs and lists (defined using pairs), including an `append` operation.
* Emulating a cyclic tag systems in lambda calculus.

[^1]: See [this Wikipedia article](https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis).

[^2]: This [blog post](https://kigawas.me/posts/y-combinator-in-python/) goes
      into the details if you are curious.
