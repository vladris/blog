# Mental Poker Part 1: Cryptography

In the [previous post](https://vladris.com/blog/2023/02/18/mental-poker-part-0-an-overview.html)
I outlined some of the interesting bits of putting together a [Mental Poker](https://en.wikipedia.org/wiki/Mental_poker)
toolkit. In this post I will talk about cryptography.

The golden rule when it comes to cryptography code is to not roll your own,
rather use something that's been battle-tested. That said, I could not find
what I needed so had to implement some stuff. I urge you not to rely on my
implementation for high-stakes poker, as it is likely buggy.

With the disclaimer out of the way, let's look at what we need to support
Mental Poker.

## Card shuffling

Recap from [this old post](https://vladris.com/blog/2021/12/11/mental-poker.html)
when I first got interested in the subject:

> Mental poker requires a commutative encryption function. If we encrypt
  $A$ using $Key_1$ then encrypting the result using $Key_2$, we should be
  able to decrypt the result back to $A$ regardless of the order of
  decryption (first with $Key_1$ and then with $Key_2$, or vice-versa).
>
> Here is how Alice and Bob play a game of mental poker:
>
> * Alice takes a deck of cards (an array), shuffles the deck, generates
    a secret key $K_A$, and encrypts each card with $K_A$.
> * Alice hands the shuffled and encrypted deck to Bob. At this point,
    Bob doesn't know what order the cards are in (since Alice encrypted
    the cards in the shuffled deck).
> * Bob takes the deck, shuffles it, generates a secret key $K_B$, and
    encrypts each card with $K_B$.
> * Bob hands the deck to Alice. At this point, neither Alice nor Bob
    know what order the cards are in. Alice got the deck back reshuffled
    and re-encrypted by Bob, so she no longer knows where each card
    ended up. Bob reshuffled an encrypted deck, so he also doesn't know
    where each card is.
>
> At this point the cards are shuffled. In order to play, Alice and Bob
  also need the capability to look at individual cards. In order to enable
  this, the following steps must happen:
>
> * Alice decrypts the shuffled deck with her secret key $K_A$. At this
    point she still doesn't know where each card is, as cards are still
    encrypted with $K_B$.
> * Alice generates a new set of secret keys, one for each card in the
    deck. Assuming a 52-card deck, she generates
    $K_{A_1} ... K_{A_{52}}$ and encrypts each card in the deck with one
    of the keys.
> * Alice hands the deck of cards to Bob. At this point, each card is
    encrypted by Bob's key, $B_K$, and one of Alice's keys, $K_{A_i}$.
> * Bob decrypts the cards using his key $K_B$. He still doesn't know
    where each card is, as now the cards are encrypted with Alice's
    keys.
> * Bob generates another set of secret keys, $K_{B_1} ... K_{B_{52}}$,
    and encrypts each card in the deck.
> * Now each card in the deck is encrypted with a unique key that only
    Alice knows and a unique key only Bob knows.
>
> If Alice wants to look at a card, she asks Bob for his key for that
  card. For example, if Alice draws the first card, encrypted with
  $K_{A_1}$ and $K_{B_1}$, she asks Bob for $K_{B_1}$. If Bob sends her
  $K_{B_1}$, she now has both keys to decrypt the card and "look" at it.
  Bob still can't decrypt it because he doesn't have $K_{A_1}$.
>
> This way, as long as both Alice and Bob agree that one of them is
  supposed to "see" a card, they exchange keys as needed to enable this.

The reason I ended up hand-rolling some cryptography is that off-the-shelf
encryption algorithms are non-commutative. With a non-commutative algorithm,
the above steps don't work: Alice cannot decrypt the deck with her secret
key $K_A$ after Bob shuffled it and encrypted it with $K_B$.

The analogy I used in [this tech talk](https://www.youtube.com/watch?v=F1gPTXAllxY)
is boxes and locks: if we have commutative encryption, we put the secret
information in a box and both Alice (using $K_A$) and Bob (using $K_B$) put
a lock on that box. It doesn't really matter in which order we unlock the two
locks - as long as both are unlocked, we can get to the content. On the other
hand, if we have non-commutative encryption, this is equivalent of Alice
putting the secret in a box locked with $K_A$, and Bob putting the whole locked
box in another box locked with $K_B$. Now Alice's key is useless while the
outerbox only has the $K_B$ lock on it.

There aren't as many applications for commutative encryption, so the popular
libraries out there provide only non-commutative encryption algorithms. The
commutative encryption algorithm we will look at is SRA.

## SRA

The SRA encryption algorithm was designed by Shamir, Rivest, and Adleman of
RSA fame. Both algorithms use their initials, but the industry-standard RSA is
non-commutative. SRA, on the other hand, is.

SRA works like this: we need a large prime number $P$. This seed prime is
shared by all players. To generate encryption keys from it, let $\phi = P - 1$.
Each player needs to find another prime $E$, such that $\phi$ and $E$ are
coprime. $E$ is that player's encryption key. The decryption key is derived
from $\phi$ and $E$ as the modulo-inverse $D$ such that
$E * D \equiv 1 \pmod{\phi}$.

To encrypt a number $N$, we raise it to $E$ modulo $P$. To decrypt an encrypted
number $N'$, we raise it to $D$ modulo $P$.

Then if player 1 encrypts a payload with $E_1$ and player 2 encrypts again
using $E_2$, the message can be decrypted by applying $D_1$ and $D_2$ in any
order. Remember, this is key to the card shuffling algorithm.

For a simple implementation, we can use arbitrarily large integers ([BigInt](https://developer.mozilla.org/en-US/docs/web/javascript/reference/global_objects/bigint)).
Unfortunately, the built-in JavaScript math libraries only work with `number`
values, so we need to implement a bit of math.

## BigInt math

First, we need to find the greatest common divisor of two numbers:

``` ts
function gcd(a: bigint, b: bigint): bigint {
    while (b) {
        [a, b] = [b, a % b];
    }

    return a;
}
```

We use this to check if two numbers are coprime (their GCD is 1).

Next, we need modulo inverse (find `x` such that `(a * x) % m == 1`). One way
of doing this is using Euclidean Division. We use the same algorithm we used
for GCD, but we keep track of the values we find at each step. Finally, if `a`
is `1`, it means there is no modulo inverse. Otherwise we find the modulo
inverse by starting with a pair of numbers `x = 1, y = 0` and iterating over
the values we found at the previous step, updating `x` to be `y` and `y` to be
`x - y * (a / b)` where `a` and `b` are values we saved from the previous step:

``` ts
function modInverse(a: bigint, m: bigint) {
    a = ((a % m) + m) % m;

    if (!a || m < 2) {
        throw new Error("Invalid input");
    }

    // Find GCD (and remember numbers at each step)
    const s = [];
    let b = m;
    while (b) {
        [a, b] = [b, a % b];
        s.push({ a, b });
    }

    if (a !== BigInt(1)) {
        throw new Error("No inverse");
    }

    // Find the inverse
    let x = BigInt(1);
    let y = BigInt(0);

    for (let i = s.length - 2; i >= 0; --i) {
        [x, y] = [y, x - y * (s[i].a / s[i].b)];
    }

    return ((y % m) + m) % m;
}
```

This gives us the modulo inverse. To recap, we use this once we have a large
prime $P$ with $\phi = P - 1$ and a large prime $E$ such that
$gcd(E, \phi) = 1$ to find our decryption key $D$.

We also need modulo exponentiation for encryption/decryption. Since we are
dealing with large numbers, we will implement exponentiation using the [ancient
Egyptian multiplication algorithm](https://en.wikipedia.org/wiki/Ancient_Egyptian_multiplication).
To raise `b` to `e` modulo `m`, if `e` is `1`, we return `b`. Otherwise we
recursively raise `(b * b) % m` to `e / 2` modulo `m`. Whenever `e` is odd,
we multiply the recursion result by an additional `b`:

``` ts
function exp(b: bigint, e: bigint, m: bigint): bigint {
    if (e === BigInt(1)) {
        return b;
    }

    let result = exp((b * b) % m, e / BigInt(2), m);

    if (e % BigInt(2) === BigInt(1)) {
        result *= b;
    }

    return result % m;
}
```

This algorithm runs in log `e` time and keeps the large numbers to a manageable
size since we apply modulo `m` at each step. We have most of the math pieces in
place. The only thing missing is a way to generate large primes.

## Generating large primes

One way of generating large primes is through trial and error: we generate a
large number, check if it is prime, and repeat if it isn't. We can generate a
large number by filling a byte array with random values, then converting it
into a `BigInt`:

``` ts
function randBigInt(sizeInBytes: number = 128): bigint {
    let buffer = new Uint8Array(sizeInBytes);
    crypto.getRandomValues(buffer);

    // Build a bigint out of the buffer
    let result = BigInt(0);
    buffer.forEach((n) => {
        result = result * BigInt(256) + BigInt(n);
    });

    return result;
}
```

This gives us a random number of as many bytes as we want (default being 128
bytes, i.e. 1024 bits). Since we are dealing with very large numbers, we can't
naively test for primality of $N$ by trying divisions up to $\sqrt{N}$, this is
too expensive. We instead use the probabilistic [Miller-Rabin test](https://en.wikipedia.org/wiki/Miller%E2%80%93Rabin_primality_test).

In short, Miller-Rabin works like this: we can write an integer $N$ (our prime
candidate) as $N = 2^S * D + 1$ where $S$ and $D$ are positive integers.

Let's take another integer $A$ coprime with $N$. $N$ is likely to be prime if
$A^D \equiv 1 \pmod{N}$ or $A^{2^{R}*D} \equiv -1 \pmod{N}$ for some
$0 <= R <= S$. If this is not the case, then $N$ is not a prime and $A$ is
called a *witness* of the compositeness of $N$.

This is a probabilistic test, so we can tell whether $N$ is for sure non-prime
or likely to be prime. Unfortunately, we can't tell for sure that $N$ is prime.
We need to run multiple iterations of this picking different $A$ values until
we are satisfied that $N$ is *likely enough* to be prime.

First, we need a helper function that checks $A$ is not a *witness* of $N$,
given $A$, $N$, and $S$ and $D$ such that $N = S^2 * D + 1$.

We compute $U$ as $A^D \pmod{N}$. If $U - 1 = 0$ or $U + 1 = N$, then $A$ is
not a witness of $N$. Otherwise, we repeat $S - 1$ times: $U = U^2 \pmod{N}$
and $A$ is not a witness if $U + 1 = N$. At this point, if we haven't confirmed
that $A$ is not a witness, we consider $A$ a witness of $N$ thus $N$ is not
prime. These are simply the checks described above ($A^D \equiv 1 \pmod{N}$
and $A^{2^{R}*D} \equiv -1 \pmod{N}$) in implementation form.

``` ts
function isNotWitness(a: bigint, d: bigint, s: bigint, n: bigint): boolean {
    if (a === BigInt(0)) {
        return true;
    }

    // u is a ^ d % n
    let u = exp(a, d, n);

    // a is not a witness if u - 1 = 0 or u + 1 = n
    if (u - BigInt(1) === BigInt(0) || u + BigInt(1) === n) {
        return true;
    }

    // Repeat s - 1 times
    for (let i = BigInt(0); i < s - BigInt(1); i++) {
        // u = u ^ 2 % n
        u = exp(u, BigInt(2), n);

        // a is not a witness if u = n - 1
        if (u + BigInt(1) === n) {
            return true;
        }
    }

    // a is a witness of n
    return false;
}
```

With this, we can finally implement Miller-Rabin. We first check a few trivial
cases (`2` and `3` are prime, even numbers are non-prime). We then find $S$ and
$D$ such that our number $N = 2^S * D + 1$ (we do this by factoring out powers
of 2 from $N - 1$).

We then repeat the test: get a random number $A < N$. If $A$ is a witness of
$N$, then $N$ is not prime. If we run this test enough times, we can safely
assume the number is prime. According to [this](https://stackoverflow.com/questions/6325576/how-many-iterations-of-rabin-miller-should-i-use-for-cryptographic-safe-primes),
40 rounds should be good enough for a 1024 bit prime.

``` ts
function millerRabinTest(candidate: bigint): boolean {
  // Handle some obvious cases
  if (candidate === BigInt(2) || candidate === BigInt(3)) {
      return true;
  }
  if (candidate % BigInt(2) === BigInt(0) || candidate < BigInt(2)) {
      return false;
  }

  // Find s and d
  let d = candidate - BigInt(1);
  let s = BigInt(0);

  while ((d & BigInt(1)) === BigInt(0)) {
      d = d >> BigInt(1);
      s++;
  }

  // Test 40 rounds.
  for (let k = 0; k < 40; k++) {
      let a = randBigInt() % candidate;

      if (!isNotWitness(a, d, s, candidate)) {
          return false;
      }
  }

  return true;
}
```

Note `d` and `s` above are technically only needed in `isNotWitness()`, but
since they are based on our prime candidate, we compute them once and pass them
as arguments to `isNotWitness()` rather than having to recompute them on each
call of the function.

We can finally implement our prime generator. We simply generate large numbers
and repeat until Miller-Rabin confirms we got a prime number:

``` ts
function randPrime(sizeInBytes: number = 128): bigint {
    let candidate = BigInt(0);

    do {
        candidate = randBigInt(sizeInBytes);
    } while (!millerRabinTest(candidate));

    return candidate;
}
```

## Cryptography

With the low-level math out of the way, we can implement the cryptography API.
First, we will define an `SRAKeyPair` as consisting of the initial large prime
$P$ and the derived $E$ and $D$ used for encryption/decryption:

``` ts
type SRAKeyPair = {
    prime: bigint;
    enc: bigint;
    dec: bigint;
};
```

We can generate a large prime using `randPrime()`. From such a prime, we can
generate an `SRAKeyPair`:

``` ts
function generateKeyPair(largePrime: bigint, size: number = 128): SRAKeyPair {
    const phi = largePrime - BigInt(1);
    let enc = BigInt(0);

    // Trial and error
    for (;;) {
        // Generate a large prime
        enc = randPrime(size);

        // Stop when generated prime and passed in prime - 1 are coprime
        if (gcd(enc, phi) === BigInt(1)) {
            break;
        }
    }

    // enc is our encryption key, now let's find dec as the mod inverse of enc
    let dec = modInverse(enc, phi);

    return {
        prime: largePrime,
        enc: enc,
        dec: dec,
    };
} 
```

If we have an `SRAKeyPair`, we can encrypt/decrypt numbers using the modulo
exponentiation function we defined above (`exp()`):

``` ts
function encryptInt(n: bigint, kp: SRAKeyPair) {
    return exp(n, kp.enc, kp.prime);
}

function decryptInt(n: bigint, kp: SRAKeyPair) {
    return exp(n, kp.dec, kp.prime);
}
```

We can also convert a string into a BigInt and vice-versa. Assuming we only
have character codes below 256 (so ASCII), we can simply encode the string
as a 256-base number where each digit is a character:

``` ts
function stringToBigInt(str: string): bigint {
    let result = BigInt(0);

    for (const c of str) {
        if (c.charCodeAt(0) > 255) {
            throw Error(`Unexpected char code ${c.charCodeAt(0)} for ${c}`);
        }

        result = result * BigInt(256) + BigInt(c.charCodeAt(0));
    }

    return result;
}
```

The ASCII assumption is reasonable, since we use this at a protocol level, not
as part of the user experience. We can decode such a number back into a string
using division and modulo:

``` ts
function bigIntToString(n: bigint): string {
    let result = "";
    let m = BigInt(0);

    while (n > 0) {
        [n, m] = [n / BigInt(256), n % BigInt(256)];
        result = String.fromCharCode(Number(m)) + result;
    }

    return result;
}
```

Now that we have these conversions, we can can implement string
encryption/decryption on top of our `encryptInt()` and `decryptInt()`
functions:

``` ts
function encryptString(clearText: string, kp: SRAKeyPair): string {
    return bigIntToString(encryptInt(stringToBigInt(clearText), kp));
}

function decryptString(cypherText: string, kp: SRAKeyPair): string {
    return bigIntToString(decryptInt(stringToBigInt(cypherText), kp));
}
```

We can encode any object as a string (and decode back strings to objects):

``` ts
function encrypt<T>(obj: T, kp: SRAKeyPair): string {
    return encryptString(JSON.stringify(obj), kp);
}

function decrypt<T>(cypherText: string, kp: SRAKeyPair): T {
    return JSON.parse(decryptString(cypherText, kp));
}
```

And that's it! We start with `randPrime()` to generate a large prime, then
use `generateKeyPair()` to derive $E$ and $D$ from it. We can then use this
`SRAKeyPair` with `encrypt()` and `decrypt()` to encrypt/decrypt objects using
the commutative SRA algorithm.

Here is a small example pulling everything together:

``` ts
// Seed prime used by both players to generate keys
const sharedPrime = randPrime();

const aliceKP = generateKeyPair(sharedPrime);
const bobKP = generateKeyPair(sharedPrime);

const card = "Ace of spades";

// Encrypt with Alice's key first, then Bob's
const aliceEncrypted = encryptString(card, aliceKP);
const aliceAndBobEncrypted = encryptString(aliceEncrypted, bobKP);

// Decrypt with Alice's key first, then Bob's
const bobEncrypted = decryptString(aliceAndBobEncrypted, aliceKP);
const decrypted = decryptString(bobEncrypted, bobKP);

// Prints "Ace of spades"
console.log(decrypted);
```

## Summary

* We went over a short overview of the SRA algorithm.
* We looked at `BigInt` implementations for GCD, modulo inverse, and modulo
  exponentiation.
* Then we generated random large numbers by filling a buffer, and testing for
  primality using the Miller-Rabin test.
* With the math in place, we implemented a key generator for SRA (takin a
  prime and deriving $E$ and $D$).
* We can encrypt/decrypt numbers by simply applying modulo exponentiation.
* We can encrypt/decrypt any string by converting it to a `BigInt`, and more
  generally any object by stringifying it.

My work-in-progress Mental Poker Toolkit is [here](https://github.com/vladris/mental-poker-toolkit).
This post covered the [cryptography package](https://github.com/vladris/mental-poker-toolkit/tree/main/packages/cryptography).
