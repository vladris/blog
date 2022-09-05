# Concepts and Modules

As a follow up to my previous post, I want to talk about two major new
C++ features that keep not making it into the standard, namely
*Concepts* and *Modules*. These would have a significant impact on the
code examples I provided while discussing dependency injection, so
here's a quick peek into the future:

## Concepts

One way to think about concepts is that concepts are to templates what
interfaces are to classes. Similarly to how interfaces specify a
contract which implementing types must satisfy, concepts specify a
contract which template argument types must satisfy. The main difference
is that interfaces/classes enable runtime polymorphism while
concepts/templates enable polymorphism at compile-time.

Runtime polymorphism:

``` c++
// Our interface
struct IEngine
{
    virtual void Start() = 0;
    virtual void Stop() = 0;
    virtual ~IEngine() = default;
};

// One implementation
class V6Engine : public IEngine
{
    void Start() override { }
    void Stop() override { }
}

// Another implementation
class V8Engine : public IEngine
{
    void Start() override { }
    void Stop() override { }
}

// A function that works against the interface
void StartEngine(IEngine& engine)
{
    engine.Start();
}

void StartEngines()
{
    V6Engine v6engine;
    V8Engine v8engine;

    // calls Start() on V6Engine instance passed as IEngine
    StartEngine(v6engine);
    // calls Start() on V8Engine instance passed as IEngine
    StartEngine(v8engine);
}
```

Here, we have a single `StartEngine` function which works against an
interface. Calling `Start()` for that interface involves a virtual
function call, which means that at runtime, given an object of a type
implementing `IEngine`, the code needs to figure out which function of
the implementing type to call. For `V6Engine`, `IEngine::Start()` is
`V6Engine::Start();` for `V8Engine`, `IEngine::Start()` is
`V8Engine::Start()`. Our classes have a vtable - a table containing this
mapping, so a virtual call looks up the actual function to call in the
vtable.

This is the "classical" object-oriented way of dispatching calls. The
advantage of this approach is that it works across binaries[^1] (we can
export `StartEngine` from a shared library and pass in an external
`IEngine` implementation to it), the disadvantage is the extra
redirection - implementing classes must have a vtable and call
resolution involves jumping through it.

Compile-time polymorphism:

``` c++
// One implementation
class V6Engine
{
    void Start() { }
    void Stop() { }
}

// Another implementation
class V8Engine
{
    void Start() { }
    void Stop() { }
}

// A generic function
template <typename TEngine>
void StartEngine(TEngine& engine)
{
    engine.Start();
}

void StartEngines()
{
    V6Engine v6engine;
    V8Engine v8engine;

    // calls Start() on V6Engine, this instantiates
    // StartEngine<V6Engine>(V6Engine& engine)
    StartEngine(v6engine);
    // calls Start() on V8Engine, this instantiates
    // StartEngine<V8Engine>(V8Engine& engine)
    StartEngine(v8engine);
}
```

A few differences to note here: we don't have an `IEngine` interface
anymore and the two types we use, `V6Engine` and `V8Engine`, no longer
have virtual functions. Calling `V6Engine::Start()` or
`V8Engine::Start()` now no longer involves a virtual call. The two
`StartEngine` calls are actually made to different functions now - at
compile time, whenever the compiler encounters a call to `StartEngine`
with a new type, it instantiates the template, meaning it creates a new
function based on that template with the template type as the provided
type. We actually end up with one function that can start a `V6Engine`
and one that can start `V8Engine`, both produced from the same template.

This is compile-time polymorphism, the advantage being that everything
is determined during build - no virtual calls etc., the disadvantage
being that the compiler needs to have a definition of the template
available whenever it needs to create a new instance[^2]. In this case
we can't encapsulate what happens inside `StartEngine` if we want
others to be able to call the function.

### With Concepts

The above works just fine, the problem being that, in general, if you
have a templated function, it's not obvious what contracts do the types
it expects need to satisfy. For example, our `StartEngine` expects that
the given type has a `Start()` function it can call. This isn't obvious
from the function declaration though. Also, compiler errors when
templates cannot be instantiated are notoriously hard to decipher. The
proposed solution to both of the above are concepts. Here is how an
engine concept would look like:

``` c++
template <typename TEngine>
concept bool Engine() {
    return requires(TEngine engine) {
        { engine.Start() };
        { engine.Stop() };
    }
}
```

This defines the `Engine` concept to require any type satisfying it to
have a `Start()` function and a `Stop()` function. `StartEngine` would
then be able to explicitly say what kind of types it expects:

``` c++
template <Engine TEngine> StartEngine(TEngine& engine)
{
    engine.Start();
}
```

It is now clear from the function declaration that `StartEngine` expects
a type satisfying the `Engine` concept. We can look at the concept
definition to see what we need to implement on our type. The compiler
would also be able to issue much clearer errors when the type we pass in
is missing one of the concept requirements.

Unfortunately, while several proposals for concepts have been put
forward in the past years, they weren't approved to be part of the
C++17 standard. That being said, it's fairly certain that they will
eventually make it into the standard.

## Modules

Another noteworthy feature are modules: currently, the `#include`
directive textually includes the given file into the source file being
compiled. This has a lot of build-time overhead (same header files get
compiled over and over as they are included in various source files) and
forces us to be extra-careful in how we scope things: what goes in a
header file vs. what goes in a source file etc.

Modules aim to replace the header/source file split and provide a better
way to group components and expose functionality. For example, here is a
header/source file pair from my previous post:

``` c++
// ICar.h
##pragma once
##include "IEngine.h"

struct ICar
{
    virtual void Drive() = 0;
    virtual ~ICar() = default;
};

std::unique_ptr<ICar> MakeCar(std::unique_ptr<IEngine> &&engine);
Car.cpp:

// Car.cpp
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

Using modules, we would have:

``` c++
module Car;

import Engine;

export struct ICar
{
    virtual void Drive() = 0;
    virtual ~ICar() = default;
};

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

export std::unique_ptr<ICar> MakeCar(std::unique_ptr<IEngine> &&engine)
{
    return std::make_unique<Car>(std::move(engine));
}
```

This is now a single file where we import the `Engine` module (instead
of `#include`), we provide the interface and concrete implementation,
the factory function, and we mark publicly-visible declarations with the
`export` keyword.

Like Concepts, Modules haven't made it into the C++17 standard, but
MSVC has a working implementation as of VS2015 Update 1.

## Dependency Injection in the Future

So putting the above together, here is how dependency injection in C++
might look like in the not too far future:

``` c++
// Engine.m
module Engine;

export template <typename TEngine>
concept bool Engine() {
    return requires(TEngine engine) {
        { engine.Start() };
        { engine.Stop() };
    }
}

// V8Engine.m
module V8Engine;

export class V8Engine
{
public:
    void Start() { // start the engine }
    void Stop() { // stop the engine }
};

// Car.m
module Car;

import Engine;
import V8Engine;

export template <Engine TEngine>
requires DefaultConstructible<TEngine>
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

// Explicit instantiation exported from this module so clients
// won't have to re-instantiate the template for V8Engine type
export class Car<V8Engine>;
```

This can be used as follows:

``` c++
import Car;
...
Car<V8Engine> car;
car.Drive();
```

This would be the equivalent of Dependency Injection with Templates I
mentioned in the previous post.

[^1]: As long as binaries are compiled with the same compiler. Otherwise
    the code produced by different compilers might have different vtable
    layouts and different name mangling.

[^2]: Other disadvantages are slower compile times and potential code
    bloat, as each template instantiation gets translated into a new
    function.
