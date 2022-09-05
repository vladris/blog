# Idris: Totality, Dependent Types, Proofs

Idris is a programming language inspired by Haskell, with a set of
innovative features to facilitate type safety and program correctness.
[Type Driven Development with
Idris](https://www.manning.com/books/type-driven-development-with-idris)
is a great introductory book which I highly recommend. In this post, I
will try to cover the features I was most impressed with, providing some
simple code samples. I will not cover syntax since most should be
familiar from Haskell. If you are not familiar with Haskell syntax, here
is a nice [syntax cheat
sheet](https://matela.com.br/pub/cheat-sheets/haskell-ucs-0.4.pdf). If
you are not interested in either Haskell or Idris syntax, start with the
last section of this post, [Thoughts about the
Future](#thoughts-about-the-future).

## Totality Checking

A total function in Idris is a function which is defined for all
possible inputs and is guaranteed to produce a result in a finite amount
of time[^1]. The compiler obviously employs a heuristic, since the
halting problem is undecidable, but is usually close enough to the truth
to guarantee correctness from this point of view. It achieves this not
by evaluating the function, but by ensuring that every recursive branch
converges to a halting branch.

Natural numbers are defined in Idris using Peano axioms, so it is easy
to prove things about them. Here is a minimal definition of natural
numbers[^2]:

``` idris
data Nat' = Z | S Nat'
```

This defines `Nat'` as a data type which can be constructed either as
`Z` (zero) or `S Nat'` (successor of another natural number). With this
definition, the compiler can easily determine the following function is
total:

``` idris
f : Nat' -> ()
f Z = ()
f (S k) = f k
```

This function return a unit given `Z`, otherwise it recursively takes
the predecessor of the argument. This converges to the `Z` case. The
following function, on the other hand, is correctly identified as
potentially non-terminating:

``` idris
g : Nat' -> ()
g n = g (S n)
```

These are trivial examples, but in general, having a compile-time check
for termination is a very powerful tool.

## Dependent Types

Dependent types are types computed from other types. To put it another
way, Idris has first-order types, meaning functions can take types as
arguments and return types as their output. Functions that compute types
are evaluated at compile time. This is similar to C++ metaprogramming,
but without employing a different syntax.

Before an example, we first need to define addition on naturals as
follows:

``` idris
(+) : Nat' -> Nat' -> Nat'
(+) Z r = r
(+) (S l) r = S (l + r)
```

Now we can declare a vector type consisting of a size (`Nat'`) and a
type argument:

``` idris
data Vect' : Nat' -> Type -> Type where
     Nil : Vect' Z a
     (::) : (x : a) -> (xs : Vect' k a) -> Vect' (S k) a
```

Here `a` is a type argument. `Vect'` has two constructors: `Nil`,
creating a `Vect'` of size `Z` containing elements of type `a` (0
elements) and `(::)`, which concatenates an object of type `a` with a
vector of size `k` of `a` and produces a vector of size `S k` containing
`a`.

Now to see dependent types in action, we can define `append'`, a
function that appends a vector to another vector:

``` idris
append' : Vect' n a -> Vect' m a -> Vect' (n + m) a
append' Nil ys = ys
append' (x :: xs) ys = x :: append' xs ys
```

The interesting part is the function signature - given a vector of size
`n` and a vector of size `m`, the resulting vector will have size
`n + m`. This information is captured in the declaration and the
compiler knows to apply the `(+)` defined above and type-check that this
is indeed true for a given pair of arguments.

## Proofs

We can also attempt to define a `reverse'` function, which recursively
appends the head of the vector to the reversed tail:

``` idris
reverse' : Vect' n a -> Vect' n a
reverse' Nil = Nil
reverse' (x :: xs) = append' (reverse' xs) [x]
```

This doesn't compile though. We get the following error message:

``` text
When checking right hand side of reverse' with expected type
        Vect' (S k) a

Type mismatch between
        Vect' (k + S Z) a (Type of append' (reverse' xs) [x])
and
        Vect' (S k) a (Expected type)

Specifically:
        Type mismatch between
                k + S Z
        and
                S k
```

We are claiming the function returns a vector of the same length as the
input vector, but we haven't proven enough theorems about our definition
of natural numbers to convince the type checker. In this particular
case, the problem is that the compiler expects an `S k` but finds an
`k + S Z`. We need to prove that these are indeed equal
(`successor of k` is the same as `k + successor of Z`). Here is the
proof:

``` idris
addOneProof : (n : Nat') -> S n = n + S Z
addOneProof Z = Refl
addOneProof (S k) = cong (addOneProof k)
```

Proofs are functions. There are a few things worth noting here: first,
the return type of this function is an equality (our theorem). Given a
natural `n`, the function proves that the equality holds. `Refl` is the
built-in reflexivity constructor, which constructs `x = x`. For the `Z`
case, we can use `Refl` to say that `S Z = Z + S Z` which is true by the
definition of `(+)`. For the `(S k)` case, we use `cong`. `cong` is a
built in function that states that equality holds after function
application. It's signature is `cong : (a = b) -> f a = f b`, which
basically means if `a` is equal to `b`, then `f a` is equal to `f b`. In
our case, we are saying that if `addOneProof k` holds, then so does
`addOneProof (S k)`, which allows us to converge on the `Z` case.

We now have a proof that `S n = n + S Z`. With this, we can prove that
the type `Vect (k + (S Z)) a` can be rewritten as `Vect (S k) a`:

``` idris
reverseProof : Vect' (k + (S Z)) a -> Vect' (S k) a
reverseProof {k} result = rewrite addOneProof k in result
```

There is some Idris-specific syntax here: `{k}` brings `k` from the
function declaration in scope, so we can refer to it in the function
body even if it is not passed in as an argument. The `rewrite ... in`
expression applies the equality in the proof above to the input, in this
case effectively rewriting `Vect (k + (S Z)) a` to `Vect (S k) a`. Note
these proofs are evaluated at compile time and simply provide
information to the type checker. With this proof, we can implement
reverse like this:

``` idris
reverse' : Vect' n a -> Vect' n a
reverse' Nil = Nil
reverse' (x :: xs) = reverseProof (append' (reverse' xs) [x])
```

This is similar to the previous implementation, we just apply
`reverseProof` to the result of `append'`. This definition compiles.

## Thoughts About the Future

Software development is generally driven by economics, where we more
often than not trade correctness for speed to market. But once the
software is up and running, correctness becomes an issue. As code
increases in complexity, the number of issues tends to increase, and the
velocity with which changes can be made without introducing regression
drops dramatically. We have various techniques that aim to maintain
stability, like automated testing, but these are not perfect: a test can
prove that for a given input we get an expected output, but cannot prove
that for *any* input we would get the expected output.

On the other hand, we have solutions that do eliminate entire classes of
issues. An example is typing. Python, Ruby, and JavaScript, all
dynamically typed, are extremely expressive and make it very easy to
whip up a proof of concept. But there is an entire class of type errors
which now turns into runtime issues. We are notoriously bad at
predicting what our code does, so the more help we get from machines to
ensure correctness, the better. In a strongly typed language, even
though it takes longer to convince the compiler that the code is
type-safe, this whole class of errors is eliminated. Language evolution
over the years tends to converge towards stronger typing: dynamic
languages are augmented with type checkers (Python has type hints,
JavaScript has TypeScript etc.) and statically typed languages are
becoming less verbose as type inference evolves. There will always be a
need for a quick prototype, but code we want to deem *reliable* should
be typed. This includes a wide range of business-critical applications
where errors are very costly.

I see Idris as the next step beyond this. Totality checking allows the
compiler to guarantee termination, eliminating hangs from the code.
First-order types allows us to push more information to the
type-checker, allowing for stricter type-checking. Proofs, expressed as
functions with regular syntax, allow the compiler to provide formal
verification of programs - here, as opposed to unit tests, we are
actually proving that we get the expected output for *any* input. These
are all tools for writing better, more correct code. As other functional
concepts got adopted over the years into more mainstream languages (for
example first-order functions, anonymous functions, algebraic types
etc.), I expect (and hope) these features to eventually be adopted too.

There is still a lot of room for improvement: writing proofs is tedious,
compiler errors are not always very clear, and, coming back to the speed
to market tradeoff, I doubt we will ever get to entire large
applications formally proven correct (barring some form of proof
inference to speed things up by a couple of orders of magnitude). That
being said, I would love to have these facilities as optional features
in other languages and at least have the ability to prove that the core
functionality of a component does what it is supposed to do, and get a
compile break whenever a regression is introduced.

Programming languages are continuously evolving and the future looks
exciting!

[^1]: Idris also supports functions that produce an infinite stream of
    values which can be used with lazy evaluation. The full definition
    of totality includes functions which don't terminate but produce
    `Inf`. This allows for non-terminating functions, but ensures
    non-termination is intentional.

[^2]: I am using `'` to avoid naming conflicts with the built-in types
    and functions. Idris already provides `Nat`, `Vect`, `append` and
    `reverse`.
