# Clean Code - Part 2

In [Part 1](https://vladris.com/blog/2016/01/07/clean-code-part-2.html) I
talked about writing less code and writing simple code. In this post I will
cover writing stateless code and writing readable code.

## Write Stateless Code

> A **pure function** is a **function** where the return value is
> only determined by its input values, without observable side effects.
> This is how **functions** in math work: `Math.cos(x)` will, for the
> same value of `x`, always return the same result.
>
> *Source:*
> [SitePoint](http://www.sitepoint.com/functional-programming-pure-functions/)

Things break when a program gets into a "bad state". There are a
couple of ways to make this less likely to happen: making data immutable
and writing functions that don't have side-effects (*pure* functions).

### Immutable data

If something shouldn't change, mark it as immutable and let the
compiler enforce that. A good rule of thumb is to mark things as `const`
(`const&`, `const*` etc.) and/or `readonly` by default, and make them
mutable only when truly needed[^1].

A simple example:

``` c++
struct StringUtil
{
    static std::string Concat1(std::string& str1, std::string& str2)
    {
        return str1 + str2;
    }

    static std::string Concat2(std::string& str1, std::string& str2)
    {
        return str1.append(str2);
    }
};
```

Both `StringUtil::Concat1` and `StringUtil::Concat2` return the same
thing for the same input, the difference being that `Concat2`, as
opposed to `Concat1`, modifies its first argument. In a bigger function,
such a change might be introduced accidentally and have unexpected
consequences down the line.

A simple way to address this is by explicitly marking the arguments as
`const`:

``` c++
struct StringUtil
{
    static std::string Concat1(const std::string& str1, std::string& str2)
    {
         return str1 + str2;
    }

    static std::string Concat2(const std::string& str1, std::string& str2)
    {
         return str1.append(str2); // Won't compile - can't call append on str1
    }
};
```

In this case, `Concat2` won't compile, so we can rely on the compiler
to eliminate this type of unintended behavior.

Another example: a simple `UpCase` function which calls `toupper` on
each character of the given string, upcasing it in place:

``` c++
void UpCase(char *string)
{
    while (*string)
    {
        *string = toupper(*string);
        string++;
    }
};
```

Calling it with

``` c++
char* myString = "Foo";
...
UpCase(myString);
```

will lead to a crash at runtime - the function will try to call
`toupper` on the characters of `myString`. The problem is that
`myString` is a `char*` to a string literal which gets compiled into the
read-only data segment of the binary. This cannot be modified.

To catch this type of errors at compile-time, we again only need to mark
the immutable data as such:

``` c++
const char* myString = "Foo";
...
UpCase(myString); // Won't compile - can't call UpCase on myString
```

In contrast with the previous example, the argument to `UpCase` is
mutable by design (the API is modifying the string in-place), but
marking `myString` as `const` tells the complier this is non-mutable
data, so it can't be used with this API.

### Pure functions

Another way to reduce states is to use pure functions. Unfortunately
there isn't a lot of syntax-level support for this in C++ and C# (C++
supports `const` member functions, which guarantee at compile time that
calling the member function on an instance of the type won't change the
attributes of that instance)[^2]

This goes back to the recommendation from Part 1 of using generic
algorithms and predicates rather than implementing raw loops. In many
cases, traversal state is encapsulated in the library algorithm or in an
iterator, and predicates ideally don't have side-effects.

``` c#
var squares = numbers.
                Where(number => number % 2 != 0).
                Select(number => number * number);
```

Above code (also from Part 1) doesn't hold any state: traversal is
handled by the Linq methods, the predicates are pure.

In general, try to encapsulate state in parts of the code built to
manage state, and keep the rest stateless. Note that immutable data and
pure functions are also an advantage in concurrent applications, since
they can't generate race conditions.

Key takeaways:

* Prefer pure functions to stateful functions and, if state is needed,
  keep it contained
* By default mark everything as `const` (or `readonly`), and only
  remove the constraint when mutability is explicitly needed

## Write Readable Code

> In computer science, the **expressive power** (also called
> **expressiveness** or expressivity) of a language is the breadth of
> ideas that can be represented and communicated in that language
> [...]
>
> * regardless of ease (theoretical expressivity)
> * **concisely and readily** (practical expressivity)
>
> *Source:*
> [Wikipedia](https://en.wikipedia.org/wiki/Expressive_power_(computer_science))

Code is read many more times than it is written/modified, so it should
be optimized for readability. What I mean by this is making the intent
of the code clear at a glance - this includes giving good descriptive
names to variables, functions, and types, adding useful comments where
appropriate (a comment should describe what the code does if it is
non-obvious; a comment like `foo(); // calls foo()` is not a useful
comment), and in general structure the code for easy reading.

For a counterexample, think back on a piece of code you read that
elicited a WTF. That's the kind of code you don't want to write.

I won't insist much here, since there are countless books and industry
best practices for improving code readability.

Another way to make the code more readable is to have a good knowledge
of the language you are using. The strength of a language lies in its
particularities, so use them whenever appropriate. This means writing
[idiomatic
code](http://stackoverflow.com/questions/84102/what-is-idiomatic-code),
which implies knowledge of the language idioms. Don't write C++ code
like C code, write it like C++ code. Don't write C# code as C++, write
it as C# etc.

Also, keep up to date on the language. Language syntax evolves to
address needs, so in general modern syntax introduces simpler, better
ways to implement things than old syntax. Take object allocation and
initialization in C++ as an example:

``` c
Foo* foo = (Foo*)malloc(sizeof(Foo));
init(foo);
...
deinit(foo);
free(foo);
```

This is the C way of allocating and initializing a structure on the
heap, then deinitializing and freeing it. Allocation and initialization
are separate steps, with opportunity to leak both memory (by omitting
the `free` call) and managed resources (by omitting the `deinit` call).
Not to mention opportunity to end up with an initialized struct (by
omitting the `init` call), or accidental double-initialization,
double-deinitialization, double-free etc.

C++ introduced classes, and the following syntax:

``` c++
Foo* foo = new Foo();
...
delete foo;
```

`new` both allocates memory and calls the constructor, while `delete`
calls the destructor then releases the memory. Many of the problems in
the C example go away, but there is still the problem of leaking the
resource by omitting the `delete` call, and the issue of calling
`delete` twice on the same memory address.

To address these issues, smart pointers were introduced in the language:

``` c++
std::shared_ptr<Foo> foo(new Foo());
```

Smart pointers encapsulate reference counting (how many `shared_ptr`
objects point to the same memory address), and automatically release the
resource when the last reference goes away. This gets rid of most
problems, but there is an even better way of allocating heap objects:

``` c++
auto foo = std::make_shared<Foo>();
```

`make_shared` has the advantage of improved performance, by allocating
memory in a single operation for both the object and the shared
pointer's own control block[^3]. It also prevents leaks due to
interleaving[^4]. So as the C++ language evolved, new constructs
appeared to address potential problems. Keeping up to date with these
updates, and incorporating them into your code will reduce the
opportunity for bugs, make the code more concise, and thus more
readable.

### Beautiful Code

I encourage you to not stop at writing *working* code, rather strive to
write *beautiful* code. I have the following quote from [Apprenticeship
Patterns](http://www.goodreads.com/book/show/5608045-apprenticeship-patterns)
on the wall behind my monitors, so I can see it while I work:

> There's always a better/faster/smarter way to do what you're
> currently doing

So don't stop as soon as something works, ask yourself *is this the
best way to implement this?*

Key takeaways:

* Come up with good names
* Write meaningful comments
* Keep up to date with your language
* Don't just write working code, write beautiful code.

## Epilogue

As I was working on putting together the talk that inspired this post, I
realized there are a few more rules of thumb which I could cover. The
current working draft is:

* Write safe code
* Write leakproof code
* Write responsive code
* Write testable code

Sometime in the future I hope to continue the series with the above, in
the meantime, I'll leave you with this one sentence summary:

> Always code as if the person who ends up maintaining your code is a
> violent psychopath who knows where you live
>
> *Source:* [Coding
> Horror](http://blog.codinghorror.com/coding-for-violent-psychopaths/)

[^1]: At the time of this writing, there is an [active
    proposals](https://github.com/dotnet/roslyn/issues/7626) to extend
    the C# language with an `immutable` keyword.

[^2]: C# has a `PureAttribue` in the `System.Diagnostics.Contracts`
    namespace (purity not compiler-enforced) and there is an [active
    proposal](https://github.com/dotnet/roslyn/issues/7561) to add a
    keyword for it too.

[^3]: This is a non-binding requirement in the standard, meaning a
    standard library implementation doesn't *have to* do this, but most
    implementations will. You can read more about it
    [here](http://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared).

[^4]: Interleaving occurs since call order is not guaranteed. For
    example, in `bar(std::share_ptr<Foo>(new foo()), baz())`, there is
    no guarantee that call order will be `new foo()`, then the shared
    pointer's constructor, then `baz()`. Calls might get interleaved
    and executed as `new foo()`, then `baz()`, then the shared pointer
    constructor, in which case an exception thrown by `baz()` would leak
    the newly allocated `Foo` object, since the shared pointer didn't
    get ownership of it yet.
