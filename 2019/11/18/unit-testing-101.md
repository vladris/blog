# Unit Testing 101

I wrote a while back about [unit
testing](https://vladris.com/blog/2017/11/09/notes-on-unit-testing.html)
from a philosophical perspective. This post is going to be more
pragmatic. My team is currently doing some MQ work which includes
improving our unit test story across the codebase. I put together a
short unit tests 101 presentation outlining some key principles:

* Run with each build.
* 100% reliability.
* Test the public interface.

## Run with Each Build

Unit tests that don't run aren't very useful. I've seen projects
before where a unit test project does exist but the tests only run if
manually executed.

The problem with this approach is that tests can stay not running for
days/weeks/months, and when they finally run, a bunch of them fail. Good
luck finding the change that introduced the regression. And wait, were
we running with that behavior all this time?

The biggest bang for the buck is making sure unit tests run as part of a
continuous integration build and pull requests get auto-rejected if a
unit test fails.

## 100% Reliability

Once unit tests run with each build, the next most important thing to
look into is ensuring they pass consistently. Flaky unit tests are bad
because there's no easy way to tell if a test run failed because of a
regression or a flaky test. Worst, if flaky tests are the standard,
engineers start ignoring the results. Hard to distill the signal from
the noise in those situations. Merge policies become more lax - after
all, we can't demand 100% green if some unit tests randomly fail.

But stepping back, when are tests flaky? When they perform IO. Hitting
the network, connecting to a database, reading a file, these are all
cases in which a transient issue outside of our control can cause a test
to fail. That's why unit tests shouldn't perform IO, rather they
should work against mocks.

Let's take, as an example, a method which performs a GET request and
logs to the console whether the request was successful:

``` c#
class Example
{
    public void Get()
    {
        var client = new HttpClient();
        var response = client.GetAsync(
            "https://www.example.com").Result;

        Console.WriteLine(response.IsSuccessStatusCode);
    }
}
```

In its current form, the method isn't really testable. Writing code
without thinking about testability yields such methods. We can refactor
this to be more testable. First, let's put all IO behind interfaces:

``` c#
interface IHttpClient
{
    Task<HttpResponseMessage> GetAsync(string url);
}

interface IOutput
{
    void WriteLine(bool value);
}
```

We can update our `Example` to use these interfaces instead of directly
working with `HttpClient` and `Console`:

``` c#
class Example
{
    private IHttpClient client;
    private IOutput output;

    public Example(IHttpClient client, IOutput output)
    {
        this.client = client;
        this.output = output;
    }

    public void Get()
    {
        var response = client.GetAsync(
            "https://www.example.com").Result;

        output.WriteLine(response.IsSuccessStatusCode);
    }
}
```

We add adapters between the interfaces and the actual implementations:

``` c#
class HttpClientWrapper : IHttpClient
{
    private HttpClient client = new HttpClient();

    public Task<HttpResponseMessage> GetAsync(string url)
        => client.GetAsync(url);
}

class ConsoleOutput : IOutput
{
    public void WriteLine(bool value)
        => Console.WriteLine(value);
}
```

With these adapters, in our production code we can put together an
instance of `Example` that works just like the original, but which is
componentized enough that we can actually test it:

``` c#
var example = new Example(
    new HttpClientWrapper(),
    new ConsoleOutput());
example.Get();
```

In our test code, we can use a framework like Moq[^1] to set up mocks
and verify that the expected calls happen:

``` c#
var mockClient = new Mock<IHttpClient>();
mockClient.Setup(
    client => client.GetAsync("https://www.example.com"))
    .Returns(Task.FromResult(
        new HttpResponseMessage {
            StatusCode = HttpStatusCode.OK
        }));

var mockOutput = new Mock<IOutput>();
mockOutput.Setup(
    output => output.WriteLine(
        It.Is<bool>(value => value == true)));

var example = new Example(
    mockClient.Object,
    mockOutput.Object);
example.Get();

mockOutput.VerifyAll();
```

The above code sets up an `IHttpClient` mock implementation which so
that when `GetAsync()` is called with the argument
`https://www.example.com` it returns a `Task<HttpResponseMessage>` with
a `StatusCode` of `HttpStatusCode.OK`. The code also sets up an
`IOutput` mock which expects a `WriteLine()` call with a `true`
argument.

We can initialize an instance of `Example` with these mocks, call
`Get()`, then verify `mockOutput` was used as expected.

### Design for Testability

The general steps for making code testable:

* Extract interface (if one doesn't exist already).
* Create adapters if concrete implementation doesn't implement an
  interface.
* Initialize class with real implementations in production.
* Initialize class with mocks in tests.
* Setup mocks to behave as required by each test.
* Verify mocks.

I will not talk about dependency injection in this post, but once all
components of the system expect several interfaces to run, it is worth
thinking about leveraging a DI framework to handle putting things
together.

With this approach, we can make any component testable except the
adapters. By their nature, our adapters perform IO. We can't reliably
test `HttpClientWrapper`. But such adapters shouldn't contain any
application logic, they should be extremely thin, simply forwarding
calls to the real implementation. It's perfectly fine to not test such
trivial code.

### Seams

Depending on the language, we can have several other ways to inject
mocks. In C++, for example, we can do it at compile-time, at link-time,
or at run-time.

At compile-time, we can use a template parameter as the "interface",
have the production version of the code instantiate it with one concrete
implementation and have the tests instantiate it with a mock:

``` c++
template <typename TImpl>
class Example
{
private:
    TImpl impl;

public:
    void Do() {
        impl.Do();
    }
};

class ConcreteImpl
{
public:
    void Do() {
        // Concrete implementation
    }
};

class MockImpl
{
public:
    void Do() {
        // Mock implementation
    }
};

// ...

Example<ConcreteImpl> ex;
```

At link-time, we can link against the concrete implementations in
production and against mock implementations in tests:

``` c++
class Example
{
private:
    Impl impl;

public:
    void Do() {
        impl.Do();
    }
};

// In concrete implementation file:
class Impl
{
public:
    void Do() {
        // Concrete implementation
    }
};

// In mock implementation file:
class Impl
{
public:
    void Do() {
        // Mock implementation
    }
};
```

At run-time, we can do something similar to our C# example above. There
are pros and cons with each approach. The run-time approach is what most
languages do, so easy to understand, though it adds more overhead. The
link-time approach is lean, but could end up being confusing: we have to
check makefiles to understand which code ends up in the binary and which
code doesn't. The compile-time approach makes the code uglier, and
requires making implementation public.

## Test The Public Interface

This one I did mention in my previous blog post to. The key point here
is that, while test frameworks usually provide various unnatural ways to
access an object's internals, tests should focus on the public members.

The public members define the "contract" that a class provides. Tests
should ensure the contract is respected and not worry about the
implementation. With this approach, the implementation can easily be
refactored and we know things still work as expected as long as all
tests pass. On the other hand, if we have tests that cover various
implementation details, they might break if we move things around, even
though the class still behaves correctly. In general, having to update
tests whenever we make tweaks to the implementation is not ideal.

The other way to look at it is that if we have some code deep in the
implementation that can't be reached through the public members, then
it is likely dead code.

## Summary

* Unit tests should run as part of continuous integration, otherwise
  they aren't really useful.
* Unit tests have to be 100% reliable. We achieve this by isolating IO
  and mocking it in tests.
* Testability recipe:
  * Code against interfaces, declare interfaces if none are
    available
  * Use thin adapters to make any concrete implementation compatible
    with any interface
  * Use concrete implementations in production and mocks in tests
* In some languages there are multiple seams where we can inject
  mocks. In C++ we can do it at compile-time, at link-time, and at
  run-time. Each has its pros and cons.
* Test the public interface not the implementation.

[^1]: Moq is my favorite C# mocking library:
    <https://github.com/moq/moq4>.
