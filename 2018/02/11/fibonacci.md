# Fibonacci

This blog post looks at a few algorithms to generate Fibonacci numbers.
For a much better treatment of these algorithms, I recommend [From
Mathematics to Generic
Programming](https://www.goodreads.com/book/show/23498372-from-mathematics-to-generic-programming).
The implementations are provided in poorly written Rust, as I'm just
learning the language.

Learning Rust and going through [The Rust Programming
Language](https://doc.rust-lang.org/book/second-edition/), I got to [the
end of chapter
3](https://doc.rust-lang.org/book/second-edition/ch03-05-control-flow.html#summary),
where there are a few simple exercises. One of them is *Generate the nth
Fibonacci number*.

## The Fibonacci Sequence

The Fibonacci sequence is defined as the sequence
$F_n = F_{n-1} + F_{n-2}$ with the seed values $F_0 = 0$ and $F_1 = 1$.

The first few values of the sequence are 0, 1, 1, 2, 3, 5, 8, 13, 21,
34, 55, 89, 144, \....

## A Naïve Algorithm

Directly translating the definition above into an algorithm to compute
the n-th Fibonacci number yields:

``` rust
fn fib(n: i32) -> i32 {
    if n == 0 { return 0; }
    if n == 1 { return 1; }

    fib(n - 1) + fib(n - 2)
}
```

This algorithm works for very small `n` values but it is extremely
inefficient as it has exponential time complexity and linear space
complexity (based on stack depth). Since
`fib(n - 1) = fib(n - 2) + fib(n - 3)`, a call like
`fib(n - 1) + fib(n - 2)` is equivalent to
`(fib(n - 2) + fib(n - 3)) + fib(n - 2)`, so the same elements of the
sequence end up being computed over and over again.

## Bottom-Up Approach

A better way to generate the nth Fibonacci number is to build it
bottom-up, starting from $F_0$ and $F_1$:

``` rust
fn fib2(n: i32) -> i32 {
    if n == 0 { return 0; }

    let mut a = 0;
    let mut b = 1;

    for _ in 1..n {
        let c = a + b;
        a = b;
        b = c;
    }

    b
}
```

Here we start with $a = F_0, b = F_1$ and with each iteration of the
loop we advance from $a = F_k, b = F_{k+1}$ to
$a = F_{k+1}, b = F_{k+2}$ until `b` becomes $F_n$.

Compared to the first algorithm, this is highly efficient, as it has
linear complexity and requires constant space. There are faster ways to
compute the nth Fibonacci number though.

## Matrix Form

The Fibonacci sequence can also be described in matrix form as follows:

$$\begin{aligned}
\begin{bmatrix}
    F_{k+2} \\
    F_{k+1}
\end{bmatrix}=\begin{bmatrix}
    1 & 1 \\
    1 & 0
\end{bmatrix}*\begin{bmatrix}
    F_{k+1} \\
    F_k
\end{bmatrix}
\end{aligned}$$

Note that the next pair of numbers in the sequence, $F_{k+3}, F_{k+2}$
can be expressed as:

$$\begin{aligned}
\begin{bmatrix}
    F_{k+3} \\
    F_{k+2}
\end{bmatrix}=\begin{bmatrix}
    1 & 1 \\
    1 & 0
\end{bmatrix}*\begin{bmatrix}
    F_{k+2} \\
    F_{k+1}
\end{bmatrix}=\begin{bmatrix}
    1 & 1 \\
    1 & 0
\end{bmatrix}*\begin{bmatrix}
    1 & 1 \\
    1 & 0
\end{bmatrix}*\begin{bmatrix}
    F_{k+1} \\
    F_k
\end{bmatrix}
\end{aligned}$$

Thus we have the formula:

$$\begin{aligned}
\begin{bmatrix}
    F_n \\
    F_n - 1
\end{bmatrix}=\begin{bmatrix}
    1 & 1 \\
    1 & 0
\end{bmatrix}^{n-1}*\begin{bmatrix}
    F_1 \\
    F_0
\end{bmatrix}
\end{aligned}$$

Since $F_0 = 0$ and $F_1 = 1$, $F_n$ is the element at index $(0, 0)$
in:

$$\begin{aligned}
\begin{bmatrix}
1 & 1 \\
1 & 0
\end{bmatrix}^{n-1}
\end{aligned}$$

Assuming we have a function for exponentiating 2x2 matrices `exp2x2`, we
can implement an algorithm to compute the nth Fibonacci number like
this:

``` rust
fn fib3(n: i32) -> i32 {
    if n == 0 { return 0; }
    if n == 1 { return 1; }

    let a = [
        [1, 1],
        [1, 0],
    ];

    exp2x2(a, n - 1)[0][0]
}
```

The complexity of this algorithm is given by the complexity of `exp2x2`.
A simple implementation of matrix exponentiation given a matrix
multiplication function `mul2x2` is:

``` rust
fn exp2x2(a: [[i32; 2]; 2], n: i32) -> [[i32; 2]; 2] {
    let mut result = [
        [1, 0],
        [0, 1],
    ];

    for _ in 0..n {
        result = mul2x2(result, a);
    }

    result
}
```

This function computes `a^n` by starting with the identity matrix and
multiplying it with a n times. The function for multiplying two 2x2
matrices is trivial:

``` rust
fn mul2x2(a: [[i32; 2]; 2], b: [[i32; 2]; 2]) -> [[i32; 2]; 2] {
    [
        [a[0][0] * b[0][0] + a[1][0] * b[0][1], a[0][0] * b[1][0] + a[1][0] * b[1][1]],
        [a[0][1] * b[0][0] + a[1][1] * b[0][1], a[0][1] * b[1][0] + a[1][1] * b[1][1]],
    ]
}
```

With `mul2x2` and `exp2x2`, our `fib3` algorithm has linear complexity,
which is determined by the number of times we call `mul2x2` in our
exponentiation function. There is a faster way to do exponentiation
though: observe that $x^7 = x^4 * x^2 * x$. In general, any number `n`
and can be decomposed as a series of powers of two. So we can implement
a `fast_exp2x2` which works as follows:

``` text
if n == 1, return a
if n is even, return fast_exp2x2(a * a, n / 2)
if n is odd, return fast_exp2x2(a * a, (n - 1) / 2) * a
```

We stop when our exponent is 1 and return `a`. If `n` is even, we square
the base and halve the exponent (for example $x^8 = (x*x)^4$). If `n` is
odd, we do the same but multiply by the base (for example
$x^9 = (x*x)^4 *
x$). This is a recursive algorithm which halves `n` at each step, so we
have logarithmic time and space complexity.

``` rust
fn fast_exp2x2(a: [[i32; 2]; 2], n: i32) -> [[i32; 2]; 2] {
    if n == 1 { return a; }

    let mut result = fast_exp2x2(mul2x2(a, a), n >> 1);

    if n & 1 == 1 {
        result = mul2x2(a, result);
    }

    result
}
```

This is a very efficient way to compute the nth Fibonacci number. But
where does this fast exponentiation algorithm come from?

## Ancient Egyptian Multiplication

The ancient Egyptian multiplication algorithm comes from the [Rhind
Papyrus](Rhind%20Mathematical%20Papyrus) from around 1500 BC. The idea
is very similar to our fast exponentiation algorithm: we can implement a
fast multiplication algorithm by relying on addition and doubling (eg.
$x * 7 = x * 4 + x * 2 + x$). The steps or our `egyptian_mul` algorithm
are:

``` text
if n == 1, return a
if n is even, return egyptian_mul(a + a, n / 2)
if n is odd, return egyptian_mul(a + a, (n - 1) / 2) + a
```

An implementation in Rust is:

``` rust
fn egyptian_mul(a: i32, n: i32) -> i32 {
    if n == 1 { return a; }

    let mut result = egyptian_mul(a + a, n >> 1);

    if n & 1 == 1 {
        result += a;
    }

    result
}
```

This multiplication algorithm relies only on addition, and multiplies
`a` by `n` in $log(n)$ steps.

`egyptian_mul` and `fast_exp2x2` algorithms have the same structure
since they are fundamentally the same: they provide an efficient way to
implement an operation defined as applying another operation n times.
Multiplication is, by definition, repeated addition. Similarly,
exponentiation is, by definition, repeated multiplication. We can
generalize these to an algorithm that given an initial value `a` of any
type `T`, an operation `op(T, T) -> T`, and `n`, the number of times to
apply `op`, provides an efficient computation using doubling and
halving:

``` rust
fn op_n_times<T, Op>(a: T, op: &Op, n: i32) -> T
    where T: Copy,
          Op: Fn(T, T) -> T {
    if n == 1 { return a; }

    let mut result = op_n_times::<T, Op>(op(a, a), &op, n >> 1);
    if n & 1 == 1 {
        result = op(a, result);
    }

    result
}
```

We can express Egyptian multiplication (`egyptian_mul`) as addition
applied `n` times:

``` rust
fn egyptian_mul(a: i32, n: i32) -> i32 {
    op_n_times(a, &std::ops::Add::add, n)
}
```

Similarly, we can express fast matrix exponentiation (`fast_exp2x2`) as
matrix multiplication applied `n` times:

``` rust
fn fast_exp2x2(a: [[i32; 2]; 2], n: i32) -> [[i32; 2]; 2] {
    op_n_times(a, &mul2x2, n)
}
```

## Industrial Strength Fibonacci

I wanted to benchmark the two interesting implementations: `fib2` and
`fib4`. The first exponential complexity implementation is highly
inefficient and even for small values of `N` (eg. `N = 50`) it takes a
long time to complete. `fib3` has linear complexity like `fib2`, but
while `fib2` just performs additions and assignments on each iteration,
`fib3` performs matrix multiplication, which is more expensive. So
`fib2` and `fib4` are more interesting to look at.

Turns out that the Fibonacci sequence grows quite fast, the 100th
Fibonacci number is `354224848179261915075`, which does not fit in an
`i32`. So let's update the implementations to use `num::BigUint`, an
arbitrary precision number. First is `fib2`:

``` rust
extern crate num;

use num::bigint::{BigUint, ToBigUint};

fn fib2(n: i32) -> BigUint {
    if n == 0 { return 0.to_biguint().unwrap(); }

    let mut a = 0.to_biguint().unwrap();
    let mut b = 1.to_biguint().unwrap();

    for _ in 1..n {
        let c = &a + &b;
        a = b;
        b = c;
    }

    b
}
```

For `fib4`, we need to update `mul2x2` to work with `BigUint` array
references, so we don't copy `BigUint` arrays:

``` rust
fn mul2x2(a: &[[BigUint; 2]; 2], b: &[[BigUint; 2]; 2]) -> [[BigUint; 2]; 2] {
    [
        [&a[0][0] * &b[0][0] + &a[1][0] * &b[0][1], &a[0][0] * &b[1][0] + &a[1][0] * &b[1][1]],
        [&a[0][1] * &b[0][0] + &a[1][1] * &b[0][1], &a[0][1] * &b[1][0] + &a[1][1] * &b[1][1]],
    ]
}
```

We also need to update our `op_n_times` so the operation now takes `&T`
instead of `T`. Note this version still works with `i32` arrays and
numbers, but now the operation is expected to take two references
instead of two values. On the other hand we no longer require that `T`
has the `Copy` trait:

``` rust
fn op_n_times<T, Op>(a: T, op: &Op, n: i32) -> T
    where Op: Fn(&T, &T) -> T {
    if n == 1 { return a; }

    let mut result = op_n_times::<T, Op>(op(&a, &a), &op, n >> 1);
    if n & 1 == 1 {
        result = op(&a, &result);
    }

    result
}
```

Then we can update our `fib4` implementation to use `BigUint`:

``` rust
fn fast_exp2x2(a: [[BigUint; 2]; 2], n: i32) -> [[BigUint; 2]; 2] {
    op_n_times(a, &mul2x2, n)
}

fn fib4(n: i32) -> BigUint {
    if n == 0 { return 0.to_biguint().unwrap(); }
    if n == 1 { return 1.to_biguint().unwrap(); }

    let a = [
        [1.to_biguint().unwrap(), 1.to_biguint().unwrap()],
        [1.to_biguint().unwrap(), 0.to_biguint().unwrap()],
    ];

    fast_exp2x2(a, n - 1)[0][0].clone()
}
```

These two implementations work with arbitrarily large numbers, for
example `fib4(10_000)` is:

<blockquote style="word-wrap:break-word">
33644764876431783266621612005107543310302148460680063906564769974680081442166662368155595513633734025582065332680836159373734790483865268263040892463056431887354544369559827491606602099884183933864652731300088830269235673613135117579297437854413752130520504347701602264758318906527890855154366159582987279682987510631200575428783453215515103870818298969791613127856265033195487140214287532698187962046936097879900350962302291026368131493195275630227837628441540360584402572114334961180023091208287046088923962328835461505776583271252546093591128203925285393434620904245248929403901706233888991085841065183173360437470737908552631764325733993712871937587746897479926305837065742830161637408969178426378624212835258112820516370298089332099905707920064367426202389783111470054074998459250360633560933883831923386783056136435351892133279732908133732642652633989763922723407882928177953580570993691049175470808931841056146322338217465637321248226383092103297701648054726243842374862411453093812206564914032751086643394517512161526545361333111314042436854805106765843493523836959653428071768775328348234345557366719731392746273629108210679280784718035329131176778924659089938635459327894523777674406192240337638674004021330343297496902028328145933418826817683893072003634795623117103101291953169794607632737589253530772552375943788434504067715555779056450443016640119462580972216729758615026968443146952034614932291105970676243268515992834709891284706740862008587135016260312071903172086094081298321581077282076353186624611278245537208532365305775956430072517744315051539600905168603220349163222640885248852433158051534849622434848299380905070483482449327453732624567755879089187190803662058009594743150052402532709746995318770724376825907419939632265984147498193609285223945039707165443156421328157688908058783183404917434556270520223564846495196112460268313970975069382648706613264507665074611512677522748621598642530711298441182622661057163515069260029861704945425047491378115154139941550671256271197133252763631939606902895650288268608362241082050562430701794976171121233066073310059947366875
</blockquote>

We can benchmark these implementations using Rust's built-in
benchmarking:

``` rust
##[bench]
fn fib4_bench(b: &mut Bencher) {
    b.iter(|| fib4(N));
}

##[bench]
fn fib2_bench(b: &mut Bencher) {
    b.iter(|| fib2(N));
}
```

On my Surface Book, for `N = 100`, we have:

``` text
test tests::fib2_bench ... bench:       7,805 ns/iter (+/- 3,042)
test tests::fib4_bench ... bench:       6,140 ns/iter (+/- 356)
```

For `N = 1_000`:

``` text
test tests::fib2_bench ... bench:      89,131 ns/iter (+/- 28,346)
test tests::fib4_bench ... bench:      16,307 ns/iter (+/- 2,087)
```

For `N = 10_000`:

``` text
test tests::fib2_bench ... bench:   2,121,140 ns/iter (+/- 132,448)
test tests::fib4_bench ... bench:     184,625 ns/iter (+/- 12,184)
```

For `N = 100_000`:

``` text
test tests::fib2_bench ... bench: 128,769,418 ns/iter (+/- 5,198,789)
test tests::fib4_bench ... bench:   7,176,026 ns/iter (+/- 364,400)
```

While `fib2` and `fib4` start with about the same performance at
`N = 100`, for `N = 100_000`, `fib4` is significantly faster. The
benchmark results don't seem to reflect the algorithmic complexity of
`fib2` (linear) and `fib4` (logarithmic), I suspect because of the
introduction of `BigUint` and operations on large numbers. Still, the
algorithm relying on fast exponentiation performs many times faster on
large Ns.

## Summary

This blog post covered:

* Algorithms to generate Fibonacci numbers: naïve recursive
  (exponential), bottom-up (linear), matrix exponentiation (linear or
  logarithmic, depending on the matrix exponentiation algorithm).
* Ancient Egyptian multiplication and fast matrix exponentiation are
  the same algorithm applied to different operations.
* A generic algorithm of efficiently applying an operation n times.
* Algorithms to generate Fibonacci numbers implemented with `BigUint`
  for arbitrary precision numbers.
* Benchmarking the `fib2` and `fib4` algorithms shows `fib4` to be
  much better as `N` increases.

My humble conclusion is that generating Fibonacci numbers is more than
an introductory exercise.
