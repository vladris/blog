# Notes on Error Handling

I recently read Joe Duffy's excellent blog post [The Error
Model](http://joeduffyblog.com/2016/02/07/the-error-model/). Joe worked
on Midori and has some great insights on error model design. I wanted to
write down a couple of personal notes on error handling.

## Using the Type System

Before even talking about error scenarios, it's worth pointing out that
there are categories of errors where the type system helps if not to
eliminate them, at least to scope them and prevent them from propagating
unchecked throughout the system.

### Arguments

In many cases, an error means the value of some variable has an invalid
value. If this invalid value is passed down to called functions, it can
manifests itself deep in the stack when it could've been caught
earlier. A simple example would be move directions for a game - let's
say the player can move `Up`, `Down`, `Left`, or `Right`. This can be
encoded as:

``` c++
const int UP = 0;
const int LEFT = 1;
const int DOWN = 2;
const int RIGHT = 3;

...

void move(int direction, player player)
{
    switch (direction)
    {
        case UP: player.move_up(); break;
        case LEFT: player.move_left(); break;
        case DOWN: player.move_down(); break;
        case RIGHT: player.move_right(); break;
        default:
            // direction should only be 0, 1, 2, 3
    }
}
```

but ultimately a caller can still pass any int value to this function
which would end up in the default branch as a direction the code
doesn't know how to handle. The alternative is, of course:

``` c++
enum class direction
{
    up,
    left,
    down,
    right
};

...

void move(direction direction, player player)
{
    ...
}
```

In this case, the type system ensures direction can only possibly hold
one of the allowed values. This is a trivial example but there are many
more interesting ones. Take for example some connection which, if
opened, can receive data and close and, if not opened, can be opened.
This can be modelled like this:

``` c++
struct connection
{
    auto open()
    {
        if (is_opened) { /* error, connection already opened */ }
        ...
    };

    auto close()
    {
        if (!is_opened) { /* error, connection not opened */ }
        ...
    }

    auto receive()
    {
        if (!is_opened) { /* error, connection not opened */ }
        ...
    }

private:
    bool is_opened;
};
```

Here we have to handle various cases where we try to perform an
open-connection operation on a connection that hasn't been opened yet,
and vice-versa. Another way to model this (as of C++17) is using a
variant and separate types for open and closed connections:

``` c++
struct opened_connection
{
    auto close() { ... }
    auto receive() { ... }
};

struct closed_connection
{
    auto open() { ... }
};

using connection = std::variant<closed_connection, opened_connection>;
```

Here, as long as we have a `closed_connection` instance, we can only
perform closed-connection operations and as long as we have an
`open_connection` instance, we can only perform opened-connection
operations. The error states we had to handle above go away as the type
system ensures we can never call `receive` on a closed connection etc.

### Return Values

The type system can also be leveraged to embellish return types as an
alternative to using return codes. For example, assume we have a
function which parses a phone number provided by the user into some
`phone_number_t` used internally. There are a few ways to implement
this:

``` c++
phone_number_t parse_phone_number(const string& input)
{
    if (phone_number_t::is_valid(input))
        return phone_number_t::from_string(input); // assume we can construct a phone_number_t from a valid string
    else
        throw invalid_phone_number { };
}
```

This is not ideal though, since exception should really be exceptional
(more on this below), and user providing invalid input should be a
completely valid scenario. The alternative would be to use a return
code:

``` c++
bool parse_phone_number(const string& input, phone_number_t& output)
{
    if (!phone_number_t::is_valid(input))
        return false;

    output = phone_number_t::from_string(input);
    return true;
}
```

This works, but calling code is uglier:

``` c++
auto phone_number = parse_phone_number(input);
```

now becomes:

``` c++
phone_number_t phone_number;
bool result = parse_phone_number(input, phone_number);
```

We can also end up in a bad state if we forget to check the return
value. The alternative is to encode the information that we either have
a `phone_number_t` or an invalid number in a type. In C++ we have (as of
C++17) `optional<T>` for this:

``` c++
optional<phone_number_t> parse_phone_number(const string& input)
{
    if (!phone_number_t::is_valid(input))
        return nullopt;
    return phone_number_t::from_string(input);
}
```

This is not quite a return error code and cannot really be ignored -
there is no implicit case from `optional<T>` to `T`, so callers need to
explicitly handle the case when the operation failed. Calling this is as
natural as the throwing version, but does not rely on exceptions[^1].
This is also called *monadic error handling* and is widely employed by
functional languages. I find this a good alternative to throwing
exceptions as long as it is well scoped and error checks don't have to
pollute too many functions in the call stack.

## Preconditions

Preconditions are conditions that should be satisfied upon entering a
function to ensure the function works as expected. When a function is
called but the preconditions are not met, it is not an error, it is a
developer bug. The recommended way of handling such a situation is, if
possible, to crash immediately. The reason for crashing is that calling
a function with preconditions not being met means the system is an
invalid state and attempting recovery is usually not worth it. Crashing
on the other hand would provide developers with dumps to help understand
how the system got into this state and fix the underlying bug.

The alternative to this is undefined behavior - calling a function
without meeting the preconditions cannot guarantee anything about the
execution of the function. Undefined behavior is used extensively
throughout the C++ standard [^2]. While failing fast is the preferred
approach, sometimes it is unfeasible to check preconditions at runtime:
for example a precondition of binary search is that it searches over an
ordered range. Performing a binary search takes logarithmic time but
validating that a range is ordered takes linear time, so adding this
check would negatively impact the run time of the algorithm. In this
case, it is OK to say that we cannot provide any guarantees on what the
function will do. Debug-time asserts are a middle ground solution, since
we can afford to perform more expensive checks in debug builds to
deterministically fail when preconditions are not met. That being said,
if the check is not prohibitively expensive, it should be performed on
all build flavors and immediately fail (via `std::terminate` or
equivalent).

What should not be done is treating such a state as an error - this is a
bug in the code and throwing an exception or returning some error result
would just leak the bug and make it impact more of the system. There
really isn't anything useful to do with such an error - it only tells
us that there is an issue in the code and we are now in a state we
should never be in. At this point we don't know which component
originated the error and we cannot deterministically recover - we might
abort the current operation but there is no guarantee that this would
bring us back to a valid state. We are in undefined behavior land, where
crashing is the best option.

## Recoverable Errors

We covered several ways to handle errors by either eliminating invalid
states at compile-time or by failing fast when in an invalid state.
There are, on the other hand, classes of errors from which we can
legitimately recover, which brings us to exception and error codes.

### Exceptions

I am a big fan of handling exceptional states using exceptions over
returning error codes. For one, the code is more readable: instead of
reserving the return type of a function to signal success or failure and
resort to out parameters, functions can be declared in a natural way. We
also end up with less code overall as instead of having to check error
codes inside all functions in the call stack in order to propagate back
an error, we simply throw it from the top of the stack and catch it
where we can deal with it. This approach also composes better - take,
for example, a generic algorithm that takes some throwing function.
Since we supply the predicate, we know what exception it can throw and
we can catch it in the code that invokes the generic algorithm, keeping
this invisible to the algorithm:

``` c++
template <typename InputIt, typename UnaryFunction>
void for_each(InputIt first, InputIt last, UnaryFunction f) noexcept(noexcept(UnaryFunction))
{
    for (; first != last; ++first)
        f(*first);
}
```

If our predicate returns an error code instead, the generic algorithm
must be aware of this:

``` c++
template <typename InputIt, typename UnaryFunction>
auto for_each(InputIt first, InputIt last, UnaryFunction f) noexcept
{
    for (; first != last; ++first)
    {
        auto result = f(*first);
        if (result != 0)
            return result;
    }
    return 0;
}
```

Note that here we are also making an assumption that 0 means success,
which is an arbitrary decision for a plug-in function.

That being said, I want to reiterate that exceptions should only be used
for exceptional cases. The readability advantage gained with exceptions
is lost if they are abused. It's great if the callee throws one or two
exception types which the caller catches and handles. On the other hand,
if we have to resort to catch-all `catch (...)` and we have so many
possible exception types coming out of a function that we can't keep
track of them, the code actually becomes harder to reason about.

An example and a counter-example: when reading a file with a set schema
generated by our application, we expect it to be in a valid format. If
it isn't, it means some data corruption occurred but this should really
be an exceptional case. If we encounter such data corruption, we can
throw an exception and let the caller handle the fact that we cannot
interpret this file. On the other hand, when reading user input, we
should never throw an exception if input is not conforming - this should
be a much more common scenario of user error.

### Return Codes

There are cases which are not exceptional enough to warrant an exception
but where some error information needs to be propagated through the call
stack. Take, for example, a compiler which encounters an invalid token
while parsing a file. Since this is user input, it should not be treated
as an exception. On the other hand, simply using an optional and failing
to parse without providing additional information is also not ideal. In
this case we probably want to return additional information around the
encountered error.

In this case we would return the error rather than throw it, but I would
still prefer an embellished type like Rust's `Result` and return an
`std::variant<T, Error>` (as of C++17). In general I consider bad
practice returning an `int` or an `HRESULT` which would afterwards have
to be decoded to understand the actual error. For simple cases, if no
other information besides success/failure has to be returned, a `bool`
would suffice, or an `enum` or `struct` which contains the expected
error information. Such an error type can be composed with a valid
return type using a variant which brings us back to monadic
error-handling.

My general rule of thumb is to use exceptions for really exceptional
situations, which keeps the code cleaner as long as the number of
exception types is managable, and use monadic error handling when errors
are expected, as long as these can be scoped to a limited number of
functions (repeated error checking all over the place is messy,
error-prone, and makes code hard to read).

## Summary

We went over various ways of handling errors:

* Declaring types that restrict the range of values a variable can
  take to eliminate invalid states at compile-time.
* Monadic error handling using embelished return types.
* Failing fast when preconditions of a function are not met.
* Throwing exceptions in exceptional cases.
* Returning strongly typed errors when errors are not exceptional.

There is still a fair amount of controversy around what is *the right
way* of handling errors. My personal take on this is that there are
tradeoffs that come with each approach and rather than saying "always
use exceptions" or "never use exceptions", it's more a matter of
choosing *the right tool for the job*. I tried to list some of the
possible approaches with their pros and cons, and how I employ them.
Your mileage may vary depending on your specific language, runtime,
problem domain, application type etc.

[^1]: This is the recommended way of [handling errors in
    Rust](https://doc.rust-lang.org/book/error-handling.html).

[^2]: See Chandler Carruth's CppCon talk [Garbage In, Garbage
    Out](https://www.youtube.com/watch?v=yG1OZ69H_-o).
