# Memory Management

Memory management involves handling memory resources allocated for a
certain task, ensuring that the memory is freed once it is no longer
needed so it can be reused for some other task. If the memory is not
freed in a timely manner, the system might run out of resources or incur
degraded performance. A memory resource that is never freed once no
longer needed is called a leak - the resource becomes unusable, usually
for the duration of the process. Another issue is *use after free*, in
which a memory resource that was already freed is used as if it wasn't.
This usually causes unexpected behavior as the code is trying to read,
modify or wrongly interpret data at a memory location. Memory management
can be *manual* - with code explicitly handling deallocation, or
*automatic*, in which memory gets freed once no longer needed by an
automated process.

## Manual Memory Management

Manual memory management is efficient, since allocations and
deallocations don't incur any overhead. In C:

``` c
typedef struct _Foo {
    ...
} Foo;

...

Foo* foo = (Foo*)malloc(sizeof(Foo));

...

free(foo);
```

The disadvantage of this approach, and the main reason automatic memory
management models were invented, is that this puts the developer in
charge of making sure memory doesn't leak and that it is not used after
it is freed. As the complexity of the code increases, this becomes
increasingly difficult. As pointers are passed around the system and get
stored in various data structures, it becomes difficult to know given
some pointer that is no longer needed whether: a) this was the very last
piece of code that actually needed to access the location pointed to by
this pointer, in which case the memory should be freed, and b) whether
the memory this pointer is pointing to is still valid and hasn't been
freed previously.

## Automatic Memory Management

Automatic memory management attempts to move the responsibility of
tracking when a memory resource is no longer needed (and handling its
deallocation) from the developer to the system. Such a system is called
*garbage collected*, as memory that is no longer needed ("garbage") is
reclaimed by the system automatically. The two most popular methods used
to automatically free memory are *tracing garbage collectors* and
*reference counting*.

### Tracing Garbage Collector

Tracing garbage collectors work by tracing references to objects on the
heap and checking whether a given resource allocated on the heap has at
least one reference path to it from the stack. If such a path exists, it
means that from the stack (an argument to a function, a local variable),
there is a way to perform a set of dereference and access the memory
resource. If such a path doesn't exist, it means the memory is
unreachable, so regardless of how executing code accesses other objects
on the heap, there is no way to access this resource - which means the
memory can be safely deallocated.

For example, a naïve tracing garbage collection algorithm,
*mark-and-sweep*, involves adding an "in-use" bit to each memory
resource allocated then, during collection, following all references
starting from the stack and marking each as "in-use". Once all used
resources are marked, the sweep stage involves walking the whole heap
and for each memory resource, if not marked as "in-use", freeing it.

Tracing garbage collectors are used by many popular runtimes, like JVM
and .NET. In C#:

``` c#
struct Bar { }

struct Foo
{
    public Bar bar;
}

...

{
    Foo foo = new Foo();
    foo.bar = new Bar();

    // there is no stack variable pointing to the Bar object, but it can
    // still be reached through foo (foo.bar), so there exists a path from
    // the stack to it, meaning code can still access it.
}

// foo goes out of scope which means neither foo nor its member Bar can be
// accessed any longer, so they can be safely collected
```

There are a couple of disadvantages with the tracing GC approach: first,
the system needs to ensure memory resources are not being allocated
while a garbage collection is taking place. This means code execution is
paused during collection, which obviously impacts performance. The
second disadvantage of this approach is that the system is not as lean
as other memory management models: memory resources are kept allocated
longer than really needed, for the time interval between the last
reference to them goes out of scope until the actual collection is
performed.

### Reference Counting

An alternative to tracing garbage collectors is reference counting. As
the name implies, a memory resource in such a system has an associated
reference count -the number of references to it. As soon as the last
reference goes out of scope, when the reference count reaches zero, the
memory can be safely deallocated. Unlike tracing, reference counting is
performed as code executes: the count of a given memory resource is
automatically increased with each assignment where the resource is on
the right-hand-side, and is automatically decreased whenever a reference
goes out of scope.

Python manages memory using reference counting:

``` python
class Foo: pass

# allocate Foo, its reference count is 1
foo1 = Foo()

# reference count is 2 after assignment
foo2 = foo1

...
# once foo1 and foo2 go out of scope, reference count becomes 0 and memory
# is automatically freed
```

C++ smart pointers work in a similar manner:

``` c++
struct Foo { };

...

// foo1 is a shared_ptr pointing to a Foo stored on the heap. Reference
// count for the Foo object is 1
auto foo1 = std::make_shared<Foo>();

// reference count becomes 2 after assignment
auto foo2 = foo1;

...
// Once foo1 and foo2 go out of scope, reference count becomes 0 and memory
// is automatically freed
```

The main advantages over tracing garbage collection are the fact that
execution doesn't need to be paused in order to reclaim memory and that
resources are deallocated as soon as they are no longer used (once
reference count becomes 0). There are also several disadvantages with
this approach: first, each memory resource needs to store an additional
reference count and updating the reference count in a multi-threaded
environment needs to be performed atomically. Second, and most
important, this memory management model does not handle *reference
cycles*.

Reference cycles occur when two heap objects hold references to each
other even after no longer being reachable from the stack. In this case,
a tracing garbage collector would mark the objects as being unreachable
and deallocate them, but simple reference counting would not be able to
identify this - from that point of view, each object is being referred
to by another object thus it should not be collected. Example of
reference cycle in Python:

``` python
class Foo: pass

a, b = Foo(), Foo()
a.other, b.other = b, a
# a.other holds a reference to b, b.other holds a reference to a
# even when a and b go out of scope, the "other" attributes still hold references
# to the objects so their reference count would not drop to 0
```

A similar example in C++:

``` c++
struct Foo
{
    std::shared_ptr<Foo> other;
};

...

auto foo1 = std::make_shared<Foo>();
auto foo2 = std::make_shared<Foo>();

foo1->other = foo2;
foo2->other = foo1;
// there are two references to each Foo object: foo1 and foo2->other for the first
// object, foo2 and foo1->other for the second object. Even if the foo1 and foo2
// variables go out of scope, neither of the objects would be collected due to the
// extra reference
```

Python and C++ solve this problem in different ways: Python supplements
reference counting with a tracing garbage collector. So while most of
the memory management is done via reference counting, a tracing garbage
collector is still employed to clean up cycles like in the above
example. This hybrid approach has he pros and cons of both of the
mechanisms discussed above. C++ avoids the execution pauses a tracing
garbage collectors would create by, instead, leveraging *weak
references*. Weak or non-owning references point to an object but do not
prevent it from being collected when all *strong* references go away.
There are several ways to express a non-owning reference, with different
advantages and drawbacks:

* A `&` reference has to be assigned on construction and cannot be
  re-assigned after being bound to an object. If used after the
  underlying object was destroyed, it causes undefined behavior.
* A `*` pointer can be `nullptr`-initialized and assigned later or
  re-assigned. Similarly, if used after the pointed-to object was
  destroyed, causes undefined behavior.
* A `weak_ptr<T>` is a standard library type implementing a non-owning
  reference. A `weak_ptr` can be converted to a `shared_ptr` (using
  its `lock()` method). If there is no strong (`shared_ptr`) reference
  to an object it gets destroyed, regardless of how many `weak_ptr`
  instances point to it. But once a `weak_ptr` successfully locks an
  object, it creates a strong reference which ensures the object is
  kept alive. The drawback of using `weak_ptr` is additional overhead:
  the control block of a smart pointer needs to store both strong and
  weak reference count (with similar atomic reference counting), and,
  even if an object gets destroyed because all strong references went
  out of scope, the control block stays alive until all weak
  references go away too.

Updating the `Foo` struct in the example above to use a `weak_ptr`
instead, the reference cycle is avoided:

``` c++
struct Foo
{
    std::weak_ptr<Foo> other;
};

...

auto foo1 = std::make_shared<Foo>();
auto foo2 = std::make_shared<Foo>();

foo1->other = foo2;
foo2->other = foo1;
// now the two Foo objects have only one strong reference to them through
// the foo1 and foo2 variables The other pointers are weak references which
// won't prevent the objects from being destroyed when foo1 and foo2 go out
// of scope
```

### Ownership and Lifetimes

An alternative way to think about heap objects is in terms of
*ownership* and *lifetime*. In this model, a heap object is uniquely
owned by some other object and gets freed automatically when the owner
is destructed. In C++, this is achieved through `unique_ptr`:

``` c++
struct Foo { };

struct Bar
{
    std::unique_ptr<Foo> foo { std::make_unique<Foo>(); }
}

...

{
    Bar bar; // this creates a Foo object on the heap, owned by bar
}
// the heap object gets freed once bar gets freed
```

Ownership of the object can be transferred by moving the `unique_ptr`.
The main advantage of this model is that it has no overhead - unlike
tracing memory which involves pausing execution or reference counting
which involves atomic count of references, a `unique_ptr` is just a
wrapper over a pointer.

Unique pointers cannot be copied though (by definition, otherwise there
would no longer denote unique ownership), so when other code needs to
access the heap object, it would need to get a reference from the owning
object:

``` c++
void UseFoo(const Foo& foo)
{
    ...
}

...

Bar bar;

UseFoo(*bar.foo);
```

The problem with this approach is that if another object ends up holding
on to a reference which outlives the owning object, the reference
becomes dangling and refers to an object which was already freed. This
becomes the equivalent of a *use after free*, so here is where the
concept of *lifetime* becomes important: none of the non-owning
references of a uniquely owned heap object should outlive the object.

Unfortunately in C++ this has to be handled through sensical design and
is mostly left up to the developer. Rust on the other hand provides
strong static analysis and lifetime annotations to ensure such issues do
not occur. In fact, the default in Rust is to have uniquely owned
objects which can be "borrowed" when needed and static analysis
ensures no dangling references appear. In C++:

``` c++
struct Foo { };

struct Bar{
    Foo* foo;
};

...

Bar bar;
{
    Foo foo;
    bar.foo = *foo;
}
// bar.foo is now a dangling pointer since Foo was freed
```

The above example used a pointer for simplicity, since a `&` reference
(`Foo&`) needs to be bound at construction time, but same applies for
that type of reference: once an object gets freed, & references and
non-owning pointers to it are left dangling. On the other hand, this
does not compile in Rust:

``` rust
struct Foo {
}

struct Bar<'a> {
    foo: &'a Foo
}

...

let bar;
{
    let foo = Foo {};
    bar = Bar { foo: &foo };
}
// compiler correctly shows `foo` dropped here while still borrowed
```

In Rust, the compiler ensures dangling references ("borrowed" objects)
do not exist once owning object goes out of scope.

It seems that in most cases, the best approach to memory management is
to use the latter model of ownership and lifetimes which comes with no
runtime overhead and handle the dangling reference problem through
static analysis. The advantages of this approach extend beyond the
runtime cost of other automatic memory management techniques to a model
which also works well in a multi-threaded environment, eg. if we only
allow the owner of an object to modify it, we can eliminate certain data
races. From a systems design perspective it is also an advantage to have
a clear understanding of ownership throughout the system.

## Summary

This post covered several memory management techniques, outlining their
pros and cons:

* Manual - human error prone.
* Automatic using a tracing garbage collector - safe but comes with
  runtime overhead.
* Automatic using reference counting - smaller runtime cost than a
  tracing garbage collector but needs additional mechanisms to deal
  with reference cycles.
* Concepts of ownership and lifetime - no runtime overhead, but should
  be supplemented by static analysis to avoid dangling references.
