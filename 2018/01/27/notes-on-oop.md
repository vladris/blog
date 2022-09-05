# Notes on OOP

I am not a huge fan of "pure" OOP. In this post I will cover a few
non-pure OOP concepts: subtyping wihtout inheritance, mixins, free
functions, and types without invariants. I will make a case for why
multi-paradigm is needed and how using a wider variety of concepts
enables us to build simpler systems.

## Duck typing

> If it walks like a duck and it quacks like a duck, then it must be a
> duck.

Let's say we have a `Duck` class. A `Duck` quacks and waddles:

``` c++
class Duck
{
public:
    void Quack() { }
    void Waddle() { }
};
```

We have a function that uses a duck:

``` c++
void foo(Duck& duck)
{
    duck.Quack();
    duck.Waddle();
}
```

The object-oriented way to implement subtyping is to inherit from the
base class:

``` c++
class UglyDuckling : public Duck { };

// ...

UglyDuckling uglyDuckling;
foo(uglyDuckling);
```

We can call `foo` on an `UglyDuckling` since `UglyDuckling` inherits
from `Duck`. We have an *is-a* relationship, so we can substitute an
`UglyDuckling` for a `Duck`. The problem with this approach is that
whenever we want something that quacks and waddles, we need to inherit
from `Duck`. More generally, this type of polymorphism is achieved by
implementing a set of interfaces like, for example, `IComparable`,
`IClonebale`, `IDisposable` and so on. This makes things slightly
complicated: what if we need something that waddles, but we don't care
about quacking? Do we separate our duck into two different interfaces?
In general, do we add an interface for each behavior and then pull
groups of interfaces together to form more refined types?

``` C++
struct IQuack
{
    virtual void Quack() = 0;
};

struct IWaddle
{
    virtual void Waddle() = 0;
};

class Duck : IQuack, IWaddle
{
public:
    void Quack() override { }
    void Waddle() override { }
};

class Penguin : IWaddle
{
public:
    void Waddle() override { }
};
```

This works, but has combinatorial complexity and we end up with deep
hierarchies which are difficult to reason about. There is another way to
achieve this though, using generic programming:

``` c++
class UglyDuckling // No inheritance
{
public:
    void Quack() { }
    void Waddle() { }
};

template <typename Duck>
void foo(Duck& duck)
{
    duck.Quack();
    duck.Waddle();
}

// ...

UglyDuckling uglyDuckling;
foo(uglyDuckling);
```

`foo` here is a templated function which only cares that the type passed
in has a `Quack` and a `Waddle` member function. There is no inheritance
involved, but we can still substitute an `UglyDuckling` for a `Duck`.
This gets us rid of all the interfaces (we don't need our `Penguin` to
explicitly implement an `IWaddle` interface, we just need it to provide
a `Waddle` member function). Our model becomes simpler - as long as a
type supports the behavior required by a function, it can be used with
that function.

## Mixins

Lore has it that multiple inheritance is bad and it is by design not
supported in Java, C#, and such. On the other hand, mixins are extremely
useful, and it is a pity that we usually have to express them via
inheritance. A mixin is a type that provides some behavior which is
*mixed in* or *included* into another type. For example, if we use
intrusive reference counting, we can isolate the reference-counting
behavior into its own type:

``` c++
class RefCounted
{
public:
    void AddRef() { ++m_refCount; }
    void Release() { if (--m_refCount == 0) { delete this; } }

    virtual ~RefCounted() = default;

private:
    std::atomic<int> m_refCount = 1;
};
```

Then we can have other types for which we want intrusive reference
counting simply mixing in this behavior:

``` c++
class Foo : public RefCounted { };
```

Now `Foo` has `AddRef` and `Release` functions which can be called by a
generic smart pointer that expects managed types to expose these member
functions. While technically `Foo` inherits from `RefCounted`,
conceptually we only care that it includes the reference counting
behavior. In such cases it is perfectly fine to mix and match and
include behavior defined across multiple other types.

## The Case for Free Functions

What is the difference between the following two `Print` functions?

``` C++
class Foo
{
public:
    void Print() { std::cout << this->Data(); }
    const char* Data() { /* ... */ }
};

void Print(const Foo& foo)
{
    std::cout << foo.Data();
}
```

The first is a member function, called with an implicit `this` argument
which points to the object instance, while the second is a free function
called with an explicit reference to a `Foo` object.

The member function approach leads to bloated objects as whenever we
need some additional processing of the type, we would have to add new
member functions. This contradicts the *Single Responsibility Principle*
which states that each class should have a single responsibility. Adding
member functions like `ToString`, `Serialize` etc. needlessly bloats a
class.

In general, we only need member functions when these functions access
private members of the type. If `Data` was private in the above example,
then the free-function version wouldn't have worked. As long as we can
implement a function that operates on a type without having to access
its private member, that function should not belong to the type.
Depending on the language, we have several options. We could put such
functions in "helper" types:

``` C#
class FooPrinter
{
    public static void Print(Foo foo) { /* ... */ }
}
```

C# provides extension methods as syntax sugar for this, which allow us
to call `foo.Print()` even though we implement the `Print` function as
an extension method:

``` C#
static class FooPrinter
{
    public static void Print(this Foo foo) { /* ... */ }
}
```

Still, the simplest thing to do is have a free function:

``` c++
void Print(const Foo& foo) { /* ... */ }
```

Being forced to group everything inside classes yields messy code. Steve
Yegge's [Kingdom of
Nouns](http://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html)
is a classic on the topic.

### Managers and Utils

Because a purely object-oriented language forces developers to think in
classes, we more often than not end up with managers and utility
classes, both being horrible replacements for free-standing functions.

Managers usually show up once we have a nice object model for the
problem space but we need to implement a set of operations on said
object model. Managers tend to be singletons. For example, we have a
`Connection` type that models a connection to a peer:

``` C#
class Connection
{
    // Open, Close, Send, Receive etc.
}
```

We also want someone to open new connections and close all opened
connections. Here is a purely object oriented `ConnectionManager`:

``` c#
class ConnectionManager
{
    private static ConnectionManager _instance = new ConnectionManager();
    private ConnectionManager() { }

    public static ConnectionManager GetInstance()
    {
        return _instance;
    }

    private List<Connection> _connections = new List<Connection>();

    public Connection Make()
    {
        var connection = new Connection();
        _connections.Add(connection);
        return connection;
    }

    public void CloseAll()
    {
        _connections.ForEach(connection => connection.Close());
    }
}
```

This maintains the list of connections and can close all of them with
a call to `CloseAll()`. Besides being verbose to use
(`ConnectionManager.GetInstance().Make()`,
`ConnectionManager.GetInstance().Close()`), this class does not make
much sense. A non-OOP implementation would look like this:

``` c++
// In .h file
class Connection
{
    // Open, Close, Send, Receive etc.
};
 
Connection& Make();
void CloseAll();
 
// In .cpp file
namespace
{
    std::vector<Connection> connections;
}
 
Connection& Make()
{
    connections.emplace_back(Connection{});
    return connections.back();
}
 
void CloseAll()
{
    for (auto&& connection : connections)
    {
        connection.Close();
    }
}
```

`Make()` and `CloseAll()` do not need to be group in some manager.
They can be free functions living next to the `Connection` type, which
is the only context within which they make sense. The list of
connections can be stored in a variable scoped to the implementation
.cpp file. "Managers" rarely make sense.

Utility classes are even worse: while a manager is usually tightly
coupled to the type it "manages", "Utils" classes end up being
dumping grounds of functions that don't seem to belong anywhere else.
The biggest problem is that each of these functions usually depends on
some other component:

``` c#
class FooUtils
{
    public static void DoBar() { /* Dependency on Bar */ }
    public static void DoBaz() { /* Dependency on Baz */ }
}
```

Now whoever takes a dependency on `FooUtils`, transitively takes a
dependency on both `Bar` and `Baz` too, even if they only really needed
one of them. If `DoBar()` and `DoBaz()` were free functions, taking a
dependency on `DoBar()` would transitively take a dependency on `Bar`
only. "Utility" types make layering a nightmare.

## When To Use Classes

I am a big believer in multi-paradigm. If our only tool is a hammer, we
can only hammer things. While pure functional languages are elegant,
they are too far removed from the machine they run on (for example we
can't implement an in-place `reverse` if all data is immutable).
Similarly, if everything is an object, we end up with too many classes
and too many complicated relationships. Procedural languages usually
provide some way to group data via `struct` or `record` types, so when
are classes useful?

The answer is *for encapsulating* - classes enable us to declare private
data and control access to it. This is useful when the class needs to
maintain invariants, which could potentially be broken if external
entities would be able to change an object's state. Let's use a `Date`
type as a made up example. Made up because dates are usually implemented
as a number representing a tick count since some set start date, and
information like *day*, *month*, and *year* is derived from that. But
let's assume we have separate *day*, *month*, and *year* fields. This
type should maintain an invariant that it represents a valid date, so we
can't have, for example, a June 31st. It's hard to enforce the
invariant with:

``` c++
struct Date
{
    uint8_t day;
    uint8_t month;
    uint8_t year;
};
```

Alternately, we can implement a class with a constructor which ensures
only valid dates can be created:

``` c++
class Date
{
public:
    Date(uint8_t year, uint8_t month, uint8_t day)
    {
        if (month == 0 || month > 12) throw /* ... */
        /* Additional checks to ensure a valid date... */
    }

    uint8_t year() const noexcept { return m_year; }
    uint8_t month() const noexcept { return m_month; }
    uint8_t day() const noexcept { return m_day; }

private:
    uint8_t m_day;
    uint8_t m_month;
    uint8_t m_year;
};
```

If we want to add an `AddDays` function, we would create a member
function [^1] which would implement the algorithm that would know when
adding a number of days would increment the month and when incrementing
the month would increment the year, such that the invariant of always
having a valid date is enforced.

On the other hand, a type which doesn't need to maintain an invariant,
say a point in the plane, should not be implemented as a class:

``` c++
struct Point
{
    int64_t x;
    int64_t y;
};
```

## Summary

Inheritance is rarely warranted, and when used, it should mostly be used
in the context of mixins - with the intention of including behavior
rather than deriving and extending. Interfaces are sometimes useful at a
component boundary, though static, template-based polymorphism is
preferred. A good design consists of a set of independent classes which
maintain invariants, and free functions that operate on them. Structure
(or record) types should be used when there is no invariant to be
maintained. Generic functions should be used when algorithms can be
generalized to multiple types as long as they satisfy some requirements
(as in the Duck Typing section above). This encourages reusable code and
systems of loosely-coupled components which can be more easily reasoned
about in isolation and reused when needed.

* Generic programming/compile-time polymorphism yields less complex
  models than inheritance
* While multiple inheritance is frowned upon, mixins provide a great
  way to add behavior to a type. The problem is including this
  behavior is usually syntactically equivalent with inheritance.
* Free functions are great. Managers and Utils are bad and should be
  avoided.
* Classes are useful when invariants need to be enforced.
  Encapsulation and member functions maintain invariants.
* A good design consists of loosely-coupled components and generic
  functions, which can be reasoned about in isolation and freely
  combined to create complex behavior.

[^1]: Or better yet a free function which takes a `Date` and returns a
    new instance - immutability seems like a good idea in this case.
