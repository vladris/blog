# Singletons

## Singletons are evil

I will start off with a word of caution that singletons should be
avoided. Singleton is the object-oriented equivalent of a global
variable -- a piece of state which anyone can grab and modify, which
makes it hard to reason locally about code and generates ugly dependency
graphs. That being said, in a large system there are situations where
there is a legitimate need for global state or some component that
exposes a service to other components.

## Implementation

[This CppCon lightning
talk](https://www.youtube.com/watch?v=23xDn3ReH7E) by Arno Lepisk covers
some implementation alternatives, suggesting that in most cases using a
namespace and flat functions is the simplest and best way to implement a
singleton:

``` c++
namespace Foo {

void DoFoo();

}
```

instead of:

``` c++
class Foo
{
public:
    void DoFoo();
};

Foo& UseFoo();
```

I completely agree with this, with the caveat that sometimes we do want
to inject dependencies and work against an interface instead of the
actual implementation, in which case the above approach might be
insufficient. Note that unless dependency injection is needed, the
default should be a namespace and flat functions.

## Dependency Injection

Given an interface, an implementation, and a function to retrieve the
singleton like the following:

``` c++
struct IFoo
{
    virtual void DoFoo() = 0;

    virtual ~IFoo() = default;
};

struct Foo : IFoo
{
    void DoFoo() override { /*...*/ };
};

IFoo& UseFoo();
```

a common mistake I see is components directly calling the function like
this:

``` c++
class Component
{
public:
    void Bar()
    {
        UseFoo().DoFoo();
    }
};
```

If the goal is to inject the dependency, for example have tests run
against `MockFoo`, this approach is not ideal. Code explicitly calls
`UseFoo` so the only way to switch implementation is to modify `UseFoo`
and provide some internal mechanism to change its return value. A better
approach is to have the client simply require an interface and provide
that at construction time:

``` c++
class Component
{
public:
    Component(IFoo& foo = UseFoo())
        : m_foo(foo)
    {
    }

    void Bar()
    {
        m_foo.DoFoo();
    }

private:
    IFoo& m_foo;
};
```

Note that in the above example we can create `Component` with a
`MockFoo` implementation of `IFoo` or some other implementation, which
is a better decoupling than directly calling `UseFoo` inside the member
functions of `Component`.

## Magic Statics

By definition, a singleton should represent a unique object, so our
`UseFoo` needs to return the same reference on each call. Ensuring that
concurrent calls from multiple threads don't cause problems was
non-trivial until C++11, which introduced "magic statics". Quote from
the C++ standard section 6.7:

> ... such a variable [with static storage] is initialized the first
> time control passes through its declaration; such a variable is
> considered initialized upon the completion of its initialization. If
> the initialization exits by throwing an exception, the initialization
> is not complete, so it will be tried again the next time control
> enters the declaration. If control enters the declaration concurrently
> while the variable is being initialized, the concurrent execution
> shall wait for completion of the initialization.

The standard now guarantees that a static would only ever be created
once, and the simple way to implement a singleton (for example according
to Scott Meyer's [Effective Modern
C++](https://www.goodreads.com/book/show/22800553-effective-modern-c))
is:

``` c++
IFoo& UseFoo()
{
    static Foo instance;
    return instance;
}
```

Or the heap-allocated version:

``` c++
IFoo& UseFoo()
{
    static auto instance = make_unique<Foo>();
    return *instance;
}
```

## Deterministic shutdown

A more interesting problem which the above implementation doesn't cover
is deterministic shutdown. A local static, once created, will be live
for the duration of the program, which might not always be desirable.
Building on the previous implementation, here is a singleton which we
can also shutdown on demand:

``` c++
template <typename T>
class Singleton
{
public:
    static T& Use()
    {
        static auto& instance = []() -> T& {
            m_instance = make_unique<T>();
            return *m_instance;
        }();

        return instance;
    }

    static void Free() noexcept
    {
        m_instance.reset();
    }

private:
    static unique_ptr<T> m_instance;
};

template <typename T> unique_ptr<T> Singleton<T>::m_instance;

/* ... */

IFoo& UseFoo()
{
    return Singleton<Foo>::Use();
}

void FreeFoo()
{
    Singleton<Foo>::Free();
}
```

Using this implementation, we can deterministically free the singleton
on demand via the `Free` function as opposed to having to wait for the
program to get unloaded, which can be useful in certain cases.

## Atomics

Magic statics provide an easy way to guarantee we end up with a single
object, but the code generated to support this is non-trivial.
Disassembly of the `UseFoo` above as compiled by Clang 4.0.0 with `-O3`
flag:

``` objdump
UseFoo():                             # @UseFoo()
        push    rbx
        mov     al, byte ptr [rip + guard variable for Singleton<Foo>::Use()::instance]
        test    al, al
        jne     .LBB0_6
        mov     edi, guard variable for Singleton<Foo>::Use()::instance
        call    __cxa_guard_acquire
        test    eax, eax
        je      .LBB0_6
        mov     edi, 8
        call    operator new(unsigned long)
        mov     qword ptr [rax], vtable for Foo+16
        mov     rdi, qword ptr [rip + Singleton<Foo>::m_instance]
        mov     qword ptr [rip + Singleton<Foo>::m_instance], rax
        test    rdi, rdi
        je      .LBB0_5
        mov     rax, qword ptr [rdi]
        call    qword ptr [rax + 8]
        mov     rax, qword ptr [rip + Singleton<Foo>::m_instance]
.LBB0_5:
        mov     qword ptr [rip + Singleton<Foo>::Use()::instance], rax
        mov     edi, guard variable for Singleton<Foo>::Use()::instance
        call    __cxa_guard_release
.LBB0_6:
        mov     rax, qword ptr [rip + Singleton<Foo>::Use()::instance]
        pop     rbx
        ret
        mov     rbx, rax
        mov     edi, guard variable for Singleton<Foo>::Use()::instance
        call    __cxa_guard_abort
        mov     rdi, rbx
        call    _Unwind_Resume
```

A lot of the generated code is the compiler implementing the guarantee
that on concurrent calls, a single intialization is performed. The
`Singleton` functions are inlined here since we are compiling with
`-O3`. We can provide a much more efficient implementation using an
atomic pointer on architectures where atomics are lock-free and we are
not worried about redundantly calling the constructor in the rare cases
of concurrent access that requires initialization:

``` c++
template <typename T>
class Singleton
{
public:
    static T& Use()
    {
        auto instance = m_instance.load();
        if (instance != nullptr)
            return *instance;

        instance = new T();
        T* temp = nullptr;
        if (m_instance.compare_exchange_strong(temp, instance))
            return *instance;

        delete instance;
        return *m_instance.load();
    }

    static void Free() noexcept
    {
        auto instance = m_instance.exchange(nullptr);
        delete instance;
    }

private:
    static atomic<T*> m_instance;
};

template <typename T> atomic<T*> Singleton<T>::m_instance;

/* ... */

IFoo& UseFoo()
{
    return Singleton<Foo>::Use();
}

void FreeFoo()
{
    Singleton<Foo>::Free();
}
```

The disassembly of the above `UseFoo` (built with the same compiler and
`-O3` flag) is:

``` objdump
UseFoo():                             # @UseFoo()
        push    rax
        mov     rcx, qword ptr [rip + Singleton<Foo>::m_instance]
        test    rcx, rcx
        jne     .LBB0_3
        mov     edi, 8
        call    operator new(unsigned long)
        mov     rcx, rax
        mov     qword ptr [rcx], vtable for Foo+16
        xor     eax, eax
        lock
        cmpxchg qword ptr [rip + Singleton<Foo>::m_instance], rcx
        je      .LBB0_3
        mov     rax, qword ptr [rcx]
        mov     rdi, rcx
        call    qword ptr [rax + 8]
        mov     rcx, qword ptr [rip + Singleton<Foo>::m_instance]
.LBB0_3:
        mov     rax, rcx
        pop     rcx
        ret
```

This code might new the object multiple times, but is guaranteed to
always return the same instance and retrieving it is more efficient than
relying on statics, since it uses a compare-exchange to guarantee
uniqueness. Many thanks to my colleague Vladimir Morozov who suggested
this approach.

## Tri-state

We now have an efficient way to create and shutdown a singleton. If
shutdown, a subsequent call to `Use` would re-create the object. One
optional feature we can add is to enforce that once shutdown, a
singleton should never be accessed again. So instead of the two-state
*not initialized* and *live*, we can use three states: *not
initialized*, *live*, *freed* and terminate if an access is attempted in
the *freed* state:

``` c++
const uintptr_t FreedSingleton = 0xDEADBEEF;

template <typename T>
class Singleton
{
public:
    static T& Use()
    {
        auto instance = Get();

        if (instance == reinterpret_cast<T*>(FreedSingleton))
            terminate();

        return *instance;
    }

    static void Free() noexcept
    {
        auto instance = m_instance.exchange(reinterpret_cast<T*>(FreedSingleton));
        delete instance;
    }

private:
    static T* Get()
    {
        auto instance = m_instance.load();
        if (instance != nullptr)
            return instance;

        instance = new T();
        T* temp = nullptr;
        if (m_instance.compare_exchange_strong(temp, instance))
            return instance;

        delete instance;
        return m_instance.load();
    }

    static atomic<T*> m_instance;
};

template <typename T> atomic<T*> Singleton<T>::m_instance;
```

We now have an efficient generic singleton which we can shutdown
on-demand and ensure clients never call after shutdown.

## Summary

* Try not to use singletons, singletons are evil.
* In most cases, a namespace and flat functions are enough, no need to
  over-complicate things.
* If dependency injection is required, make sure dependency is
  properly injected during construction as opposed to member functions
  directly calling the singleton-retrieving function.
* Magic statics provide an easy way to implement singletons.
* Atomics are more efficient than magic statics when they are
  lock-free and we aren't worried about potentially having multiple
  constructor calls in race cases.
* If needed, singletons can be extended with a shutdown mechanism.
* Three-state singletons can terminate on use-after-shutdown.
