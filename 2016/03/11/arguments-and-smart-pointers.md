# Arguments and Smart Pointers

For efficiency reasons, C++ had a myriad of ways to pass data around. A
function can take arguments in several forms:

``` c++
void foo(T);
void foo(T*);
void foo(T&);
void foo(T**);
void foo(T&&);
```

No to mention adding `const` and smart pointers into the mix.

And speaking of pointers, we have raw pointers, `unique_ptr`,
`shared_ptr,` `CComPtr`/`ComPtr` (for Windows COM objects), and, if you
are working in older codebases, `auto_ptr`, and maybe even some
homebrewed refcounted pointers.

All of this might seem a bit daunting and make C++ seem more complicated
than it really is. One way to think about it is from the ownership
perspective: objects are resources and the key question is "who owns
this resource?". This should dictate both how a particular resource is
allocated (and released) and the shape function arguments should take.
Lifetime is also an important consideration - a resource shouldn't be
released while other components expect it to be available, but it should
be released as soon as it is no longer needed.

In this post I will try to cover the various ways in which resources can
be allocated, owned, and passed around.

## Stack Objects and Passing Arguments

The simplest, clearest thing to do is allocate objects on the stack. A
stack object doesn't involve any pointers:

``` c++
void foo()
{
    Bar bar;
}
```

The variable `bar` of type `Bar` is created on the stack. Once the stack
frame is popped (once the function is done executing, either through a
normal return or due to an exception) the object goes away. This is the
easiest, safest thing to do. The only reasons we wouldn't always do
this are time and space requirements: lifetime-wise, we might want to
somehow use `bar` after `foo()` returns - for example we might want to
pass it around to some other object that wants to use at a later time;
in terms of space, stack memory is more limited than the heap, so large
objects are better kept on the heap to avoid overflow.

### Pass by value

One way to get around the lifetime requirement is to pass the object by
value:

``` c++
void do_suff(Bar bar); // hands bar off to some other object

void foo()
{
    Bar bar;
    do_suff(bar);
}
```

Let's assume for this example that `do_suff` takes the argument and
sticks it into some global object which will use it as some future time.

The above code will simply create a copy of the object, so whatever
`do_suff` gets won't be the original resource which gets freed once the
function returns, rather a copy of it. Copying an object costs both run
time and space, but if neither are a big concern, this is a great, safe
way of ensuring resources don't get released before we're done with
them.

### Move

C++11 introduces a cheaper way of achieving this, through move
semantics:

``` c++
void do_suff(Bar&& bar); // hands bar off to some other object

void foo()
{
    Bar bar;
    do_suff(std::move(bar));
}
```

With move semantics, the resource is actually *moved* into the `do_suff`
function. Once this happens, the original object is left in an undefined
state and shouldn't be used anymore. This approach is usually employed
when we have a sink argument - we *sink* `bar` to its final resting
place somewhere and `foo()` no longer cares about it after passing it
down to `do_stuff`.

One thing to keep in mind is that `move` is not magic, so `Bar` needs to
declare a [move
constructor](http://en.cppreference.com/w/cpp/language/move_constructor)
in order for this to do what we expect it to do. If Bar doesn't declare
a move constructor, the above becomes a simple copy[^1].

### Pass by reference

On the flipside, when we care about the size, so that we don't want to
create a copy of the object, but we aren't worried about the lifetime -
meaning the object we pass to `do_stuff` won't have to outlive the
function call, we can pass by reference:

``` c++
void do_suff(const Bar& bar); // bar is only used within do_suff

void foo()
{
    Bar bar;
    do_suff(bar);
}
```

Note the `const` above - this means `do_suff` will use `bar` but won't
modify it. By default, arguments should be marked as `const` unless the
function does indeed need to alter the object. Regardless of constness,
in this case we pass a reference to `bar` as an argument, which is very
cheap (a reference has the same size as a pointer). The only caveat is
that `do_stuff` should not pass this to some other object that outlives
the function call (eg. a global object which tries to use it later),
because as soon as `foo` returns, the reference becomes invalid.

### Pass by pointer

A pointer argument would look like this:

``` c++
void do_stuff(const Bar* bar);

void foo()
{
    Bar bar;
    do_stuff(&bar);
}
```

A good rule of thumb is to not do this. The difference between passing
by reference and by pointer in this case is that a pointer can be null,
while a reference can't. So passing by pointer here automatically
brings the need to perform null checks to ensure bad things don't
happen. You would need to make a very good argument to convince me
during code review that using a pointer instead of a reference is
appropriate. Unless working against a legacy API which can't be
changed, I highly discourage use of raw pointers.

### Summary

In summary, when designing an API:

1. Take argument by value if copying it is not a concern
2. Take argument by `const&` if it's not a sink argument, meaning we
   don't need to refer to it passed the function call
3. Take argument by reference (`&`) if 2) but the API needs to modify
   it
4. Take argument by `&&` if it's a sink argument, the type has a move
   constructor, and copying it is expensive
5. Don't pass raw pointers around

## Heap Objects and Smart Pointers

In all of the examples above, `bar` was an object created on the stack.
This works great in some cases, but some objects are simply too big to
fit on the stack, or it doesn't make sense for them to do so (if, for
example, we want to vary their size at runtime). In this case, we
allocate the object on the heap and keep a pointer to it.

Once we start working with heap objects, ownership becomes even more
important: unlike stack objects, which get automatically destroyed when
their stack frame gets popped, heap objects need to be explicitly
deleted. This responsibility should be with the *owner* of the object.

### Unique pointer

A unique pointer (`std::unique_ptr`) is a wrapper around a raw pointer
which will automatically delete the heap object when it goes out of
scope itself:

``` c++
void foo()
{
    auto ptrBar = std::make_unique<Bar>();
} // ptrBar goes out of scope => heap object gets deleted
```

The above call to `make_unique` allocates an instance of Bar on the heap
and wraps the pointer to it into the unique pointer `ptrBar`. Now
`ptrBar` *owns* the object and as soon as ptrBar goes out of scope, the
heap object is also deleted.

Unique pointers cannot be copied, so we can never accidentally have more
than one single `unique_ptr` pointing to the same heap object:

``` c++
auto ptrBar = std::make_unique<Bar>();
...
std::unique_ptr<Bar> ptrBar2 = ptrBar; // Won't compile
```

Of course, if we *really* want to, we can get the raw pointer out of
`ptrBar` using `get()` and we can initialize a `unique_ptr` from a raw
pointer -

``` c++
// Please don't do this
std::unique_ptr<Bar> ptrBar2(ptrBar.get())
```

but this is very bad - now both pointers think they have sole ownership
of the resource, and as soon as one goes out of scope, using the other
one leads to undefined behavior. In general, the same way there are very
few good reasons to use raw pointers, there are very few good reasons to
call `get()` on a smart pointer.

### Shared pointer

Sometimes, we do need to have several pointers pointing to the same heap
object. In this case, we can use shared pointers. Shared pointers
pointing to the same heap object keep a common reference count. Whenever
a new shared pointer is created for that particular heap object, the
reference count is incremented. Whenever a shared pointer for that heap
object goes out of scope, the reference count is decremented. Once the
last shared pointer goes out of scope, the heap object is deleted.

``` c++
void foo()
{
    auto ptrBar1 = std::make_shared<Bar>();
    // one pointer to a Bar object on the heap (ref count = 1)
    {
        auto ptrBar2(ptrBar1);
        // second shared pointer (ref count = 2)
    }
    // ptrBar2 goes out of scope (ref count = 1)
}
// ptrBar1 goes out of scope (ref count = 0) => heap object is deleted
```

Shared pointers incur a bit more overhead than unique pointers -
reference counting needs to be atomic to account for multi-threaded
environments, which comes with a runtime cost. The reference count
itself also needs to be stored somewhere, which is a small space cost.
Unique pointers don't have these time and space costs since they don't
need to count references - there is always only one pointer to the
object.

Costs aside, shared pointers also don't make the ownership clear -
there are several instances "owning" the heap resource at the same
time, which can potentially alter it and step on each other's toes. In
general, prefer unique pointers to shared pointers whenever possible.

### Raw pointer

Avoid using raw pointers. Raw pointers don't express ownership, so they
don't offer the same guarantees that a) the resource pointed to gets
properly cleaned up and b) the resource pointed to is still valid at a
given time. This leads to dereferencing invalid memory and
double-deletes (trying to free the same heap object multiple times),
which means undefined behavior. Also, don't mix smart and raw
pointers - the smart pointers will keep doing their job happily, with
the potential of making the raw pointers invalid.

### COM pointers

On Windows, COM uses a different reference counting mechanism: the base
`IUnknown` interface declares `AddRef` and `Release` methods, which
implementations are expected to use to keep track of the reference
count. `CComPtr` (in ATL) and `ComPtr` (in WRL) are the COM smart
pointers. They call `AddRef` and `Release` on the owned object, and the
owned object is supposed to delete itself once its reference count drops
to 0. Note that COM uses a slightly different mechanism than the
standard library shared pointers -instead of the smart pointer keeping
track of the reference count in the control block and deleting the
object once the last reference goes away, COM objects are expected to
keep track of their reference count themselves through the `AddRef` and
`Release` methods and self-delete when the last reference goes away
(through `Release` call). The COM smart pointers only need to call
`Release` when they go out of scope.

It's not a good idea to have both standard library and COM pointers
point to the same object, as each might decide to delete the object at
different times -`shared_ptr` looks at the `shared_ptr` refcount while
COM objects look at their internal reference count. So a `shared_ptr`
might decide to delete an object while a `ComPtr` still expects it to be
valid or vice-versa. In general, when working with COM objects, use COM
smart pointers.

### auto_ptr

`auto_ptr` is a deprecated smart pointer. Unless working with an old
compiler and standard library, use `unique_ptr` or `shared_ptr` instead.

### Other smart pointers

Old code bases might have custom smart pointer implementations, for the
simple fact that automatic memory management is always a good idea, and
there is C++ code that predates the introduction of smart pointers into
the standard library. When interoperating with legacy code, use whatever
works, but when writing new code, do prefer standard library smart
pointers to homebrewed ones.

### Summary

In summary, when creating objects:

1. Create them on the stack if feasible (note that standard library
   types like `std::vector` and `std::string` internally keep their
   data on the heap, but they fit perfectly well on the stack, so you
   don't need to create an `std::vector` on the heap just because you
   are planning to store a lot of elements in it - the vector manages a
   heap array internally already).
2. Use a `unique_ptr` when creating them on the heap, to make ownership
   obvious.
3. Use a `shared_ptr` only when `unique_ptr` isn't sufficient (review
   your design first, might be a design issue).
4. Use COM smart pointers like `CComPtr` when dealing with COM.
5. Don't use `auto_ptr` or other old constructs unless working with
   legacy code/compiler.
6. Don't use raw pointers.

## Passing Smart Pointers as Arguments

We covered passing arguments and smart pointers. Now combining the two,
how do we pass heap objects as arguments? Turns out Herb Sutter has [a
great post](http://bit.ly/227Na5c) on this exact topic on his blog. I
can't hope to explain better than him, so go read his post. I will try
to summarize:

### Pass by reference the pointed-to type

Rather than forcing callers to use `unique_ptr` or `shared_ptr` by
specifying the smart pointer type (which makes assumptions about
ownership), just ask for a reference to the pointed-to-type:

``` c++
void do_stuff(const Bar& bar);

void foo()
{
    auto ptrBar = std::make_unique<Bar>();
    do_stuff(*ptrBar);
}
```

Herb also mentions raw pointer to the underlying type if the argument
can be null, but as I mentioned above, I'd rather stick to references
and discourage use of raw pointers as a general rule of thumb.

### Pass smart pointer by value

Passing a `unique_ptr` by value implies a sink argument - since a
`unique_ptr` cannot be copied, it has to be `std::move`'d in.
Interestingly, Scott Meyers has [a post](http://bit.ly/1WgkvEh) on his
blog where he disagrees with this and argues that arguments of move-only
types should be specified as `&&`:

``` c++
void do_stuff(unique_ptr<Bar>&& ptrBar); // sink

void foo()
{
    auto ptrBar = std::make_unique<Bar>();
    do_stuff(std::move(ptrBar));
}
```

Passing a `shared_ptr` by value implies the function wants to partake in
the ownership - in other words, will keep somewhere a reference to the
object after the function returns, but unlike the above `unique_ptr`
example, it won't have exclusive ownership of the resource:

``` c++
void do_stuff(shared_ptr<Bar> ptrBar);

void foo()
{
    auto ptrBar = std::make_shared<Bar>();
    do_stuff(ptrBar); // copy-constructs another shared_ptr which shares ownership of the heap object
}
```

### Pass smart pointer by reference

Only expect a smart pointer by non-const reference if the function is
going to modify the smart pointer itself (eg. by making it point to a
different object). In my experience, this is a rare occurrence.

``` c++
// Implies this function modifies the pointer itself.
void do_stuff(shared_ptr<Bar>& ptrBar);
```

There is no good reason to expect a `const&` to a `unique_ptr`, just
reference the underlying type:

``` c++
// void do_stuff(const unique_ptr<Bar>& ptrBar);
// No reason to use the above as opposed to
void do_stuff(const Bar& bar);
```

Expect `const&` to `shared_ptr` only if the function *might* create a
copy of the smart pointer. If the function would never create a copy of
the pointer, simply use `&` to underlying type. If the function would
always copy the pointer, expect `shared_ptr` by value.

``` c++
// Might or might not share ownership
void do_stuff(const shared_ptr<Bar>& ptrBar);

// Will never share ownership
void do_stuff(const Bar& bar);

// Will always share ownership
void do_stuff(shared_ptr<Bar> ptrBar);
```

### Summary

A summary of the summary:

1. Take argument by `&` to underlying type if you only care about the
   heap object, not about the pointer.
2. Take `unqiue_ptr` by `&&` to transfer ownership.
3. Take `shared_ptr` argument by value to partake in ownership.
4. Take smart pointer by (non-const) reference only if you are going to
   modify the smart pointer itself.
5. No need for `const&` to `unique_ptr` (just take `&` to underlying
   type)
6. Take `const&` to `shared_ptr` only if unknown whether function wants
   ownership (take by `&` to underlying type if function never wants
   ownership, `shared_ptr` by value if function always wants
   ownership).

[^1]: Move constructors can be implicitly declared by the compiler if
    certain conditions are met, see
    [here](http://en.cppreference.com/w/cpp/language/move_constructor#Implicitly-declared_move_constructor)
    for details.
