# IoC Containers

In this post I will go over the basics of IoC containers and walk
through a very simple C# implementation.

## Motivation

An inversion of control container is a component that encapsulates
dependency injection and lifetime management of other components[^1].
Assume we have some well componentized code where classes work against
interfaces:

``` c#
interface IFoo
{
    void DoFoo();
}

class Foo : IFoo
{
    public void DoFoo() { }
}

class Bar
{
    // Coded against IFoo interface, so decoupled from
    // concrete Foo implementation
    public Bar(IFoo foo)
    {
        foo.DoFoo();
    }
}
```

In order to create an instance of `Bar`, we need to pass it some `IFoo`
object. We could do it by `new`-ing up a `Foo` object at the call site
of `Bar`'s constructor:

``` c#
var bar = new Bar(new Foo());
```

This doesn't scale very well though. Most classes require more than one
dependency, so we can't realistically litter the code with:

``` c#
var a = new A(new B(new C(), new D()), new E(new D()));
```

Factory functions address part of the problem. We can have, for example:

``` c#
public IA MakeIA()
{
    return new A(MakeIB(), MakeIE());
}
```

Another advantage of using factory functions instead of directly calling
constructors is that a factory can encapsulate lifetime of objects.
Changing from a new instance on every call to a singleton can be
encapsulated in the factory so callers of `MakeIA` don't need to
change.

We would then call `MakeIA` whenever we needed an `IA` instance. But
where do these factories belong? They do not belong with the concrete
types they are implementing, because having a static `MakeIA` on class
`A` would still require callers to reference `A` (as in `A.MakeIA()`).
Since these factories become the only places in the system where
knowledge of which type resolves to which interface, it makes sense to
keep them together:

``` c#
static class Factories
{
    public static IA MakeIA()
    {
        return new A(MakeIB(), MakeIE());
    }

    public static IB MakeIB()
    {
        return new B(MakeIC(), MakeID());
    }

    public static IC MakeIC()
    {
        return new C();
    }

    public static ID MakeID()
    {
        return new D();
    }

    public static IE MakeIE()
    {
        return new E(MakeID());
    }

    // ...
}
```

This works pretty well: changing which concrete type binds to an
interface becomes a changed scoped to one of the factories. But
maintaining this by hand can become tedious. The good news is it can be
automated, which is what an IoC container does.

## A Basic IoC Container

A very basic container would be able to bind a concrete implementation
against an interface and return an instance of the concrete
implementation when asked for an interface:

``` c#
class Container
{
    public static void Register<T>(/* ... */)
    {
        // ...
    }

    public static T Resolve<T>()
    {
        // ...
    }
}
```

The simplest possible thing to pass to `Register` is a factory function,
in which case our container would have to maintain a mapping from type
to factory:

``` c#
class Container
{
    private static Dictionary<Type, Func<object>> _registeredTypes = new Dictionary<Type, Func<object>>();

    public static void Register<T>(Func<T> factory) where T : class
    {
        _registeredTypes[typeof(T)] = factory;
    }

    public static T Resolve<T>()
    {
        return (T)_registeredTypes[typeof(T)]();
    }
}
```

This is how it can be used:

``` c#
Container.Register<IA>(() => new A(Container.Resolve<IB>(), Container.Resolve<IE>()));
Container.Register<IB>(() => new B(Container.Resolve<IC>(), Container.Resolve<ID>()));
Container.Register<IC>(() => new C());
Container.Register<ID>(() => new D());
Container.Register<IE>(() => new E(Container.Resolve<ID>()));

// ...

var a = Container.Resolve<IA>();
```

This is fine, but still requires a lot of hand-maintenance. One of the
main features of a container is the ability to use reflection and
resolve some of these dependencies automatically. Given a type, we can
find its first public constructor by calling `GetConstructor` on it:

``` c#
foreach (var param in typeof(A).GetConstructors().First().GetParameters())
{
    Console.WriteLine(param);
}
```

So given a type, we should be able to generate a factory function for
it. A simple way of doing it is by calling `Invoke` on the retrieved
constructor and attempting to retrieve all its arguments from the
container:

``` c#
public static void Register<T>(Type type)
{
    var constructor = type.GetConstructors().First();

    _registeredTypes[typeof(T)] = () =>
    {
        return constructor.Invoke(constructor.GetParameters().Select(
            param => _registeredTypes[param.ParameterType]()
        ).ToArray());
    };
}
```

Now registering the interfaces becomes a lot easier:

``` c#
Container.Register<IA>(typeof(A));
Container.Register<IB>(typeof(B));
Container.Register<IC>(typeof(C));
Container.Register<ID>(typeof(D));
Container.Register<IE>(typeof(E));

// ...

var a = Container.Resolve<IA>();
```

It's usually a good idea to support registration by both type and
factory function, for cases where the construction is more involved or
the types of the constructor arguments, for various reasons, are not
registered with the container.

### Efficient Construction

Calling `Invoke` on a `ConstructorInfo` is notoriously slow[^2]. There
are several strategies to make this invocation faster. One of them is by
using `System.Linq.Expressions`, which are a set of types that help
declare and compile lambdas at runtime:

``` c#
public static void Register<T>(Type type)
{
    var constructor = type.GetConstructors().First();

    _registeredTypes[typeof(T)] = (Func<object>)Expression.Lambda(
        Expression.New(constructor, constructor.GetParameters().Select(
            param =>
            {
                Func<object> resolve = () => _registeredTypes[param.ParameterType]();
                return Expression.Convert(
                    Expression.Call(Expression.Constant(resolve.Target), resolve.Method),
                    param.ParameterType);
            }
        ))).Compile();
}
```

The above implementation compiles a lambda which is equivalent to the
`Invoke` logic. There are several other techniques to dynamically
generate functions, including `Reflection.Emit` and
`System.Runtime.CompilerServices`. Another decision point is whether
resolution is done lazily or not. The above implementation is lazy,
resolving each constructor parameter does not require an entry for it in
the container when this particular lambda is compiled. The relevant line
is:

``` c#
Func<object> resolve = () => _registeredTypes[param.ParameterType]();
```

If we were to replace this with:

``` c#
Func<object> resolve = _registeredTypes[param.ParameterType];
```

it would fail to compile the lambda when registering `A` unless all
other dependencies are already in the container. This approach is
flexible, in that type bindings can be resolved at runtime, but can
incur a bit more overhead. An alternative would be to register all types
with the container first, then generate the factories in a separate
step. In that case, for each type, we could map out exactly what calls
need to be made to set it up based on information already available to
the container. Such an implementation gets more complex, so I won't go
into the details, but worth noting that it is possible.

## Lifetimes

Containers also encapsulate lifetime management. The most basic
non-instance lifetime is singleton, which means a unique instance during
the lifetime of the app. Let's extend our container to also support
resolving singletons. First we need a way to wrap a factory into a
function that only calls it once, then caches the result:

``` c#
private static Func<object> SingletonDecorator(Func<object> factory)
{
    var instance = new Lazy<object>(factory);
    return () => instance.Value;
}
```

This relies on `Lazy` to ensure uniqueness. Now we can enable singleton
registrations for factories and types:

``` c#
public static void MarkSingleton<T>()
{
    _registeredTypes[typeof(T)] = SingletonDecorator(_registeredTypes[typeof(T)]);
}
```

This effectively decorated the registered factory with the singleton
logic. There are various other lifetimes an object could require, for
example: threaded (where instances are cached per thread, so the same
instance is always returned on the same thread but not across threads),
scoped (where there is an API to mark beginning and end of a scope
within which the same instance is always returned, but another one gets
created in another scope) etc.

## A Note on Loading

One interesting observation made while profiling a .NET application is
that a container usually forces the loading of all referenced assembly.
The .NET runtime defers assembly loading until a method is called which
references a type in a not-yet-loaded assembly. This forces assembly
loading as the runtime needs the metadata of the type. When using an IoC
container, all types are usually registered as soon as the application
boots, in which case all assemblies get pulled in during registration
time (as opposed to on-demand at a later time).

## Resources

The complete source code for the container in this blog post is:

``` c#
public class Container
{
    private static Dictionary<Type, Func<object>> _registeredTypes = new Dictionary<Type, Func<object>>();

    public static void Register<T>(Func<T> factory) where T : class
    {
        _registeredTypes[typeof(T)] = factory;
    }

    public static void Register<T>(Type type)
    {
        var constructor = type.GetConstructors().First();

        _registeredTypes[typeof(T)] = (Func<object>)Expression.Lambda(
            Expression.New(constructor, constructor.GetParameters().Select(
                param =>
                {
                    Func<object> resolve = () => _registeredTypes[param.ParameterType]();
                    return Expression.Convert(
                        Expression.Call(Expression.Constant(resolve.Target), resolve.Method),
                        param.ParameterType);
                }
            ))).Compile();
    }

    public static void MarkSingleton<T>()
    {
        _registeredTypes[typeof(T)] = SingletonDecorator(_registeredTypes[typeof(T)]);
    }

    public static T Resolve<T>()
    {
        return (T)_registeredTypes[typeof(T)]();
    }

    private static Func<object> SingletonDecorator(Func<object> factory)
    {
        var instance = new Lazy<object>(factory);
        return () => instance.Value;
    }
}
```

I also recently open-sourced a minimal container
[here](https://github.com/microsoft/minioc). That implementation
includes support for scoped lifetimes, but otherwise it is still very
minimal. It was used in a couple of small projects where constraints
were around size/dependent assemblies rather than feature richness.

For a popular open source container with many more features, check out
[AutoFac](https://autofac.org/).

## Summary

In this post we went over a few IoC container basics:

* Motivation for containers.
* A primitive container supporting factory registration.
* Using reflection to support type registration.
* Approaches to implementing constructor calls: `Inove`,
  `Linq.Expressions`, others. Lazy resolution vs. generating
  constructor calls in a separate step.
* Lifetime management and a singleton implementation.

[^1]: For a much more detailed treatment, see Martin Fowler's
    [article](https://www.martinfowler.com/articles/injection.html)

[^2]: An interesting benchmark on [Stack
    Overflow](https://stackoverflow.com/questions/35805609/performance-of-expression-compile-vs-lambda-direct-vs-virtual-calls)
