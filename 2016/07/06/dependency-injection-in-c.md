# Dependency Injection in C++

In this post, I will switch gears from functional C++ to object oriented
C++ and talk about dependency injection.

Let's start with a simple example: take a `Car` class with a `Drive()`
method. Let's say this class contains a `V8Engine` attribute with
`Start()` and `Stop()` methods. An initial implementation might look
like this:

*V8Engine.h (publicly visible)*:

``` c++
##pragma once

class V8Engine
{
public:
    void Start();
    void Stop();
};
```

*V8Engine.cpp*:

``` c++
##include "V8Engine.h"

V8Engine::Start()
{
    // start the engine
}

V8Engine::Stop()
{
    // stop the engine
}
```

*Car.h (publicly visible)*:

``` c++
##pragma once
##include "V8Engine.h"

class Car
{
public:
    void Drive();

private:
    V8Engine m_engine;
};
```

*Car.cpp*:

``` c++
##include "Car.h"

void Car::Drive()
{
    m_engine.Start();
    // drive
    m_engine.Stop();
}
```

## Dependency Injection with Interfaces

In the above example, `Car` is tightly coupled to `V8Engine`, meaning we
can't create a car without a concrete engine implementation. If we want
the ability to swap various engines or use a mock engine during testing,
we could reverse the dependency by creating an `IEngine` interface and
decoupling `Car` from the concrete `V8Engine` implementation. This way,
we only expose an `IEngine` interface and a factory function. `Car` can
work against that:

*IEngine.h (publicly visible)*:

``` c++
##pragma once

struct IEngine
{
    virtual void Start() = 0;
    virtual void Stop() = 0;
    virtual ~IEngine() = default;
};

std::unique_ptr<IEngine> MakeV8Engine();
```

*V8Engine.cpp*:

``` c++
##include "IEngine.h"

class V8Engine : public IEngine
{
    void Start() override { /* start the engine */ }
    void Stop() override { /* stop the engine */ }
};

std::unique_ptr<IEngine> MakeV8Engine()
{
    return std::make_unique<V8Engine>();
}
```

*Car.h (publicly visible)*:

``` c++
##pragma once
##include "IEngine.h"

class Car
{
public:
    Car(std::unique_ptr<IEngine>&& engine)
        : m_engine(std::move(engine))
    {
    }

    void Drive();
private:
    std::unique_ptr<IEngine> m_engine;
};
```

*Car.cpp*:

``` c++
##include "Car.h"

void Car::Drive()
{
    m_engine->Start();
    // drive
    m_engine->End();
}
```

## Notes

### A note on headers

Headers simply get textually included in each compilation unit by the
`#include` directive. It is not mandatory to provide a header file for
each class declaration. If a class can be scoped to a single source
file, then it doesn't need a header declaration (for example the
`V8Engine` class above does not need a V8Engine.h header corresponding
to the V8Engine.cpp). It is also a good idea to have public headers and
internal headers: public headers contain the public API surface and can
be included by other parts of the system, while internal headers are
only used within the component and should not be included by external
code.

Default should be the least visible: try to keep everything inside the
cpp file (like V8Engine.cpp). If that is not enough, an internal header
might do. A declartion should be pulled into a public header only when
external components need to reference it.

### A note on interfaces

It's a good idea to declare a default virtual destructor: if a deriving
type has a destructor, it won't get called if we store an upcast
pointer to the interface unless the interface declares a virtual
destructor. Note a destructor does not to be expicitly defined -
compiler might generate a default one.

MSVC compiler provides a `__declspec(novtable)`[^1] custom attribute
which tells the compiler not to generate a vtable for pure abstract
classes. This reduces code size. Below is the `IEngine` declaration with
this attribute:

``` c++
struct __declspec(novtable) IEngine
{
    virtual void Start() = 0;
    virtual void Stop() = 0;
    virtual ~IEngine() = default;
};
```

I won't include it in the code samples in this post, but it's worth
keeping in mind when working with MSVC.

### A note on factory functions

When working with interfaces as opposed to concrete types, we use
factory functions to get object instances. Below is a possible naming
convention, taking object ownership into account:

``` c++
std::unique_ptr<IFoo> MakeFoo();
IFoo& UseFoo();
std::shared_ptr<IFoo> GetFoo();
```

The first function, `MakeFoo`, returns a unique pointer, passing
ownership to the caller. Like in the example above, the `unqiue_ptr` can
be moved into the object, which ends up owning it. Use a Make when each
call creates a new instance.

The second function implies there already exists an `IFoo` object which
is owned by someone else, with the guarantee that it will outlive the
caller. In that case, there is no need for pointers and we can simply
return a reference to the object. This can be used, for example, for
singletons. Below is an example of a singleton `Engine`:

``` c++
IEngine& UseEngine()
{
    static auto instance = std::make_unique<Engine>();
    return *instance;
}
```

The third function, `GetFoo`, implies shared ownership - we get an
object that other objects might hold a reference to, but we don't have
the lifetime guarantee a singleton would give us, so we need to use a
shared pointer to make sure the object is kept alive long enough.

## Mocking

Since `Car` now works with an `IEngine` interface, in test code we can
mock the engine:

*Test.cpp*:

``` c++
##include "Car.h"

class MockEngine : public IEngine
{
    void Start() override { /* mock logic */ }
    void Stop() override { /* mock logic */ }
};

void Test()
{
    Car car(std::make_unique<MockEngine>());

    // Test Car without a real Engine
}
```

We can also expose `Car` as a simple interface, hiding its
implementation details, in which case we would end up with the
following:

*ICar.h (publicly visible)*:

``` c++
##pragma once
##include "IEngine.h"

struct ICar
{
    virtual void Drive() = 0;
    virtual ~ICar() = default;
};

std::unique_ptr<ICar> MakeCar(std::unique_ptr<IEngine> &&engine);
```

*Car.cpp*:

``` c++
##include "ICar.h"

class Car : public ICar
{
public:
    Car(std::unique_ptr<IEngine>&& engine)
         : m_engine(std::move(engine))
    {
    }

    void Drive() override
    {
         m_engine->Start();
         // drive
         m_engine->Stop();
    }

private:
    std::unique_ptr<IEngine> m_engine;
};

std::unique_ptr<ICar> MakeCar(std::unique_ptr<IEngine>&& engine)
{
    return std::make_unique<Car>(std::move(engine));
}
```

Test would become:

``` c++
##include "ICar.h"

class MockEngine : public IEngine
{
    void Start() override { /* mock logic */ }
    void Stop() override { /* mock logic */ }
};

void Test()
{
    auto car = MakeCar(std::make_unique<MockEngine>());

    // Test ICar without a real Engine
}
```

Note this allows the caller to pass in any `IEngine`. We provide an
out-of-the-box `V8Engine` but other engines can be injected when `Car`
gets constructed. The headers IEngine.h and ICar.h are public per our
above defintion.

In general, it's great if we can get the rest of the component code and
unit tests to work against the interface. Sometimes though we might need
to know more about the actual implementation inside our component, even
if externally we only expose an interface. In that case, we can add an
internal Car.h header:

*Car.h (internal)*:

``` c++
##pragma once
##include "ICar.h"

class Car : public ICar
{
public:
    Car(std::unique_ptr<IEngine>&& engine)
         : m_engine(std::move(engine))
    {
    }

    void Drive() override;

private:
    std::unique_ptr<IEngine> m_engine;
};
```

*Car.cpp* becomes:

``` c++
##include "Car.h"

void Car::Drive()
{
    m_engine.Start();
    // drive
    m_engine.Stop();
}

std::unique_ptr<ICar> MakeCar(std::unique_ptr<IEngine>&& engine)
{
    return std::make_unique<Car>(std::move(engine));
}
```

Now we can include the internal header, and, while not necessarily
recommended, we can cast `ICar` to `Car` inside the component:

``` c++
auto icar = MakeCar(MakeV8Engine());
auto& car = static_cast<Car&>(*car);
```

Another trick if needing access to internals (again, not something
necessarily recommended), is to make the unit test class testing `Car` a
friend of the `Car` class, in which case it can access its private
members.

In summary, with this approach we are able to:

* Hide implementation details in the .cpp files
* Work against abstract interfaces
* Inject dependencies during object construction

## Dependecy Injection with Templates

An alternative to the above is to use templates. In this case, we would
have to provide the implementation inside the header file, as code needs
to be available when templates get instantiated:

*V8Engine.h (publicly visible)*:

``` c++
##pragma once

class V8Engine
{
public:
    void Start();
    void Stop();
};
```

*V8Engine.cpp*:

``` c++
##include "V8Engine.h"

V8Engine::Start()
{
    // start the engine
}

V8Engine::Stop()
{
    // stop the engine
}
```

*Car.h (publicly visible)*:

``` c++
##pragma once

template <typename TEngine>
class Car
{
public:
    void Drive()
    {
        m_engine.Start();
        // drive
        m_engine.Stop();
    }

private:
    TEngine m_engine;
};
```

Note `Car` is implemented in the header and `V8Engine` is also a
publicly visible header. Now we can create an instance of `Car` like
this:

``` c++
##include "V8Engine.h"
##include "Car.h"

...

Car<V8Engine> car;
```

Mocking the engine in test code would look like this:

``` c++
##include "Car.h"

class MockEngine
{
    void Start() { /* mock logic */ }
    void Stop() { /* mock logic */ }
};

void Test()
{
    Car<MockEngine> car;

    // Test Car without a real Engine
}
```

With this approach we are able to:

* Inject dependencies during template instantiation
* No need for virtual calls (note `TEngine` is not an interface, so
  calls can be resolved at compile-time)
* `Car<T>` can be default-constructed

A drawback here is we expose the implementation details of `Car` inside
the header file and we have to make this publicly visible.

## Hybrid Approach

We can use a hybrid approach if we don't need an externally injected
`Engine`. Say our component provides a `V8Engine`, a `V6Engine`, and we
have a `MockEngine` used during testing. We have the same
componentization requirements but don't need to expose all the details
to consumers. In that case we could have something like this:

*ICar.h (publicly visible)*:

``` c++
##pragma once

struct ICar
{
    virtual void Drive() = 0;
    virtual ~ICar() = default;
};

std::unique_ptr<ICar> MakeV8Car();
std::unique_ptr<ICar> MakeV6Car();
```

*Car.h (internal)*:

``` c++
##pragma once
##include "ICar.h"

template <typename TEngine>
class Car : public ICar
{
public:
    void Drive() override
    {
        m_engine.Start();
        // drive
        m_engine.Stop();
    }

private:
    TEngine m_engine;
};
```

*Car.cpp*:

``` c++
##include "Car.h"
##include "V8Engine.h"
##include "V6Engine.h"

std::unique_ptr<ICar> MakeV8Car()
{
    return std::make_unique<Car<V8Engine>>();
}

std::unique_ptr<ICar> MakeV6Car();
{
    return std::make_unique<Car<V6Engine>>();
}
```

Test would remain the same as in the example above, where we worked
against a `Car` type (not an `ICar`) which we instantiate with a
`MockEngine`.

With this approach:

* Our external API is an interface
* Internally we still inject the dependency using a template

With this approach, we do have an interface and virtual calls for `Car`
but not for `TEngine` types. One drawback with this approach is that
consumers cannot inject their own Engine type: we can only create cars
with engines that are known within our component.

## Summary

We decoupled `Car` from `V8Engine` and looked at three ways of injecting
the dependency:

* Using interfaces, where dependency is injected at runtime during
  object creation
* Using templates, where dependency is injected at compile-time during
  template instantiation
* A hybrid approach which uses templates internally but exposes only
  interfaces publicly

Each of these approaches has pros and cons, the tradeoffs mostly being
around encapsulation (how much of the component code we expose
publicly), runtime (templates are instantiated at compile-time so no
virtual calls etc.), type constraints (with templates we don't require
engines to implement a particular `IEngine` interface), and flexibility
(with the hybrid approach we can't inject an external engine, we can
only use what the component has available internally).

[^1]: For more details on `novtable`, see
    [MSDN](https://msdn.microsoft.com/en-us/library/k13k85ky.aspx).
