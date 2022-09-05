# Implementing a Variant Type in C&#35;

A variant, or discriminated union type[^1], is a type that can hold a
value of any of a finite set of types. For example, a
`Variant<int, string>` can hold either an `int` or a `string` value.
This is also known as a *sum type*, as its domain is the sum of the
`int` and `string` domains. Contrast this with a `Tuple<int, string>`,
also known as a *product type*, which holds *both* an `int` and a
`string` (so its domain is the product of the `int` and `string`
domains).

First, let's look at how something like this would be achieved without a
`Variant` type. Let's take an expression tree where a node can be either
an `int` value or an expression consisting of an operation (let's say
addition and multiplication) and two operands which are in turn nodes.
We could implement this by starting with an `INode` base interface and
deriving our types from that:

``` c#
enum Op { Add, Mul }

interface INode { }

class ExpressionNode : INode
{
    public Op Op { get; set; }
    public INode Left { get; set; }
    public INode Right { get; set; }
}

class ValueNode : INode
{
    public int Value { get; set; }
}
```

This works, but has a couple of drawbacks -- first, to discriminate
between the types allowed to be part of the tree and the types that
aren't, we need to establish a typing relationship and force every type
of node in our tree to implement a dummy `INode` interface. In the case
of a value, even though we just need an `int`, we must wrap it into a
`ValueNode` class because `int` itself does not implement `INode`.

Another drawback is that in many cases we want to restrict the types
that can participate in our system (in this case our expression tree).
This is harder to enforce via an interface, as one could always
implement some other `class FooNode : INode` and there is no
compile-time way to prevent this node from becoming part of our tree.

This is how the above tree would be declared if we had a
`Variant<T1, T2>`:

``` c#
enum Op { Add, Mul }

class Expression
{
    public Op Op { get; set; }
    public Variant<Expression, int> Left { get; set; }
    public Variant<Expression, int> Right { get; set; }
}
```

First, we no longer need an `INode`, as it would get replaced by
`Variant<Expression, int>`, which effectively translates to *it's either
an Expression or an int*. There is also no room to sneak in another type
without being explicit about it. We do not need to wrap `int` into
another class either just to make it conform to our hierarchy, as a
`Variant` can handle it directly.

So how would we go about designing such a generic variant in C#?

## Design Considerations

Our variant implementation should satisfy a few requirements:

* Since C# does not support variadic generic arguments, we want
  implementations from `Variant<T1>`, holding a value of a single
  type, up to `Variant<T1, T2, T3, T4, T5, T6, T7, T8>`, holding a
  value of any of 8 types. This is in line with how other library
  types like `Tuple` are implemented.

* We want to support implicit casting from one of the generic types to
  the variant, as this enables assignment:

  ``` c#
  Variant<string, int> variant = 42;
  ```

* We want to support explicit casting from a variant to any of its
  generic types. Since a variant might hold a value of a different
  type, we should be explicit in this case, as a mismatch between the
  actual held value and the cast-to type would throw an
  `InvalidCastException`.

* We need an API to get the value of the variant as a given type. A
  type mismatch between the given type and the one actually held by
  the variant would result in an `InvalidCastException`:

  ``` c#
  Variant<string, int> variant = 42;

  int value = variant.Get<int>();
  string value = variant.Get<string>(); // throws InvalidCastException
  ```

* We need an API to check if the value of the variant is of a certain
  type:

  ``` c#
  Variant<string, int> variant = 42;

  variant.Is<string>(); // false
  variant.Is<int>(); // true
  ```

* We need to support variants where the same type appears several
  times, like `Variant<int, int>`. There are legitimate use cases for
  this, for example an API that would return either an error code or a
  value, both being of the same type. For such cases, we need another
  way to explicitly set a value as the first, second, and so on type,
  and an `Index` property that would tell us which occurrence of the
  type it is (as `Get<int>() called on a Variant<int, int>` would
  succeed in returning us an `int`, but we wouldn't be able to tell
  whether it got in there as a `T1 or as a T2`.

* We would also provide a non-generic `Get()` which returns an
  `object`, so we can use pattern matching on a variant:

  ``` c#
  Variant<string, int> variant = 42;

  switch (variant.Get())
  {
      case string s:
          Console.WriteLine("It's a string: " + s);
          break;
      case int i:
          Console.WriteLine("It's an int: " + i);
          break;
  }
  ```

* We need to override equality: two variants are equivalent if they
  are the same type (same generic parameters), they contain values of
  the same type at the same index, and the contained values are
  equivalent:

  ``` c#
  Variant<string, int> variant1 = 42;
  Variant<string, int> variant2 = 42;

  variant1.Equals(variant2); // true
  ```

Given these requirements, let's see how an implementation would look
like.

## Implementation

### API

We'll start with a `Variant<T1, T2>` and build up from there. Adding
more generic arguments becomes easy once this implementation is figured
out. Starting from the simpler `Variant<T1>` would not uncover some of
the issues mentioned above, like the need for an index and ability to
handle `T1` and `T2` being the same type. Let's define our API based on
the requirements:

``` c#
public sealed class Variant<T1, T2>
{
    // Variant API
    public byte Index
        => throw new NotImplementedException();

    public bool Is<T>()
        => throw new NotImplementedException();

    public T Get<T>()
        => throw new NotImplementedException();

    public object Get()
        => throw new NotImplementedException();

    // T1 constructor, casts
    public Variant(T1 item)
        => throw new NotImplementedException();

    public static implicit operator Variant<T1, T2>(T1 item)
        => throw new NotImplementedException();

    public static explicit operator T1(Variant<T1, T2> item)
        => throw new NotImplementedException();

    // T2 constructor, casts
    public Variant(T2 item)
        => throw new NotImplementedException();

    public static implicit operator Variant<T1, T2>(T2 item)
        => throw new NotImplementedException();

    public static explicit operator T2(Variant<T1, T2> item)
        => throw new NotImplementedException();

    // Equality
    public override bool Equals(object obj)
        => throw new NotImplementedException();

    public override int GetHashCode()
        => throw new NotImplementedException();
}
```

We have the `Index` property which should be `0` if the variant is
holding a `T1` and `1` if the variant is holding a `T2`. We're using
0-based indexing for this, though it is a bit awkward that generic
arguments start, by convention, from 1. This is in line with what other
.NET types do, for example `Tuple` provides a 0-based indexer.

`It<T>()` allows callers to check if the variant is currently holding a
`T`, while `Get<T>()` should return a `T` or throw an
`InvalidCastException`. The non-generic version `Get()` simply returns
an `object`.

Below that, for both `T1` and `T2` we provide a constructor which takes
a `T1` (or `T2`) and places it in the variant, implicit casts from `T1`
and `T2` to `Variant<T1, T2>`, and explicit casts the other way around.

Finally, we override `Equals(object)` and `GetHashCode()` (it's always a
good idea to override `GetHashCode` when overriding `Equals`).

### Type erasure

Let's look at how we would store a value. Unlike a `Tuple<T1, T2>`, we
don't want to store both a `T1` *and* a `T2`, rather we want either a
`T1` *or* a `T2`. In order to generalize this, we need a way to perform
type-erasure, which means a way to store any type (as we want a generic
implementation), while at the same type we need to keep track of the
stored type so we can answer `Is<T>()` properly. Let's create a
`VariantHolder` to handle this.

We could achieve this by storing everything as an `object`
(type-erasure) and a `Type` (for type information), like this:

``` c#
sealed class VariantHolder
{
    public object Item { get; }

    private Type _itemType { get; }

    public bool Is<T>() => typeof(T) == _itemType;

    public object Get() => Item;

    public VariantHolder(object item)
    {
        Item = item;
        _itemType = item.GetType();
    }
}
```

We can then implement our variant in terms of this:

``` c#
public sealed class Variant<T1, T2>
{
    private VariantHolder variant;

    public bool Is<T>() => variant.Is<T>();

    public T Get<T>() => (T)variant.Get();

    public object Get() => variant.Get();

    public Variant(T1 item)
        => variant = new VariantHolder(item);

    public static implicit operator Variant<T1, T2>(T1 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T1(Variant<T1, T2> item)
        => item.Get<T1>();

    /* ...
       Same for T2, ignoring Equals() and GetHashCode() for now */
}
```

This implementation is not quite correct, as it stores too much type
information:

``` c#
class Base { }
class Derived : Base { }

/* ... */

Variant<Base, int> variant = new Derived();
variant.Is<Base>(); // == false!
```

We are comparing the actual type of the item, though we should store it
as one of the generic types of the variant declaration. A better idea is
to make our `VariantHolder` itself generic:

``` c#
sealed class VariantHolder<T>
{
    public T Item { get; }

    public bool Is<U>() => typeof(U) == typeof(T);

    public object Get() => Item;

    public VariantHolder(T item) => Item = item;
}
```

This gets us rid of the extra `_itemType` member (we can use `typeof(T)`
on the generic parameter), but we have another problem: how do we
declare this in our `Variant`? If we make it a `VariantHolder<T1>`, then
we won't be able to store a `T2` value and vice-versa. There is a way
around this - we can extract an interface:

``` c#
interface IVariantHolder
{
    bool Is<T>();

    object Get();
}
```

We can declare that our `VariantHolder<T>` implements this interface (it
already does, as it has both a generic `Is` and a non-generic `Get`:

``` c#
sealed class VariantHolder<T> : IVariantHolder
{
    public T Item { get; }

    public bool Is<U>() => typeof(U) == typeof(T);

    public object Get() => Item;

    public VariantHolder(T item) => Item = item;
}
```

And now we can implement our `Variant` in terms of an `IVariantHolder`:

``` c#
public sealed class Variant<T1, T2>
{
    private IVariantHolder variant;

    public bool Is<T>() => variant.Is<T>();

    public T Get<T>() => ((VariantHolder<T>)variant).Item;

    public object Get() => variant.Get();

    // T1 constructor, casts
    public Variant(T1 item)
        => variant = new VariantHolder<T1>(item);

    public static implicit operator Variant<T1, T2>(T1 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T1(Variant<T1, T2> item)
        => item.Get<T1>();

    // T2 constructor, casts
    public Variant(T2 item)
        => variant = new VariantHolder<T2>(item);

    public static implicit operator Variant<T1, T2>(T2 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T2(Variant<T1, T2> item)
        => item.Get<T2>();

    /* ...
       Ignoring Equals() and GetHashCode() for now */
}
```

For `Get<T>()` we use a cast to `VariantHolder<T>` as opposed to
`(T)_variantHolder.Get()` as this avoids an extra boxing operation if
`T` is a value type. This correctly throws `InvalidCastException` if
called with the wrong type. If we wanted to throw a different exception
or add more details to the exception, we could either wrap this cast in
a try/catch and catch an `InvalidCastException` or we could check the
type using `Is<T>()` before performing the cast.

### Index and disambiguation

The only problem with this implementation is that we cannot instantiate
a variant if `T1` and `T2` are the same:

``` c#
Variant<int, int> variant = 42;
```

yields a compiler error: "Ambiguous user defined conversions". If we
try calling the constructor:

``` c#
var variant = new Variant<int, int>(42);
```

we get "The call is ambiguous between the following methods\...". If
`T1` and `T2` are the same type, there is no way to disambiguate between
constructors and casts. Because of this, we need to add our `Index`
property and provide a way to explicitly construct the variant with an
index. First, let's add `Index` to our current implementation:

``` c#
public sealed class Variant<T1, T2>
{
    private IVariantHolder variant;

    public byte Index { get; }

    public bool Is<T>() => variant.Is<T>();

    public T Get<T>() => ((VariantHolder<T>)variant).Item;

    public object Get() => variant.Get();

    private Variant(IVariantHolder item, byte index)
    {
        variant = item;
        Index = index;
    }

    public Variant(T1 item)
        : this(new VariantHolder<T1>(item), 0)
    { }

    public static implicit operator Variant<T1, T2>(T1 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T1(Variant<T1, T2> item)
        => item.Get<T1>();

    public Variant(T2 item)
        : this(new VariantHolder<T2>(item), 1)
    { }

    public static implicit operator Variant<T1, T2>(T2 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T2(Variant<T1, T2> item)
        => item.Get<T2>();

    /* ...
       Ignoring Equals() and GetHashCode() for now */
}
```

We added a read-only `Index` property, a private constructor that not
only sets the `IVariantHolder` but also the `Index`, and we updated our
two constructors, `Variant(T1 item)` and `Variant(T2 item)` to
internally call this private constructor with the correct index.

Now we have an `Index` property which accurately keeps track of the
index of the type stored, so for a `Variant<int, int>` we would be able
to tell whether we set the first or second `int`, but we still can't
disambiguate between constructor calls if `T1` and `T2` are the same. We
can solve this by adding a couple of explicit factory methods:

``` c#
public static Variant<T1, T2> Make1(T1 item)
    => new Variant<T1, T2>(new VariantHolder<T1>(item), 0);

public static Variant<T1, T2> Make2(T2 item)
    => new Variant<T1, T2>(new VariantHolder<T2>(item), 1);
```

We could have provided a way for the callers to explicitly provide an
index, but it becomes hard to enforce that the index is in sync with the
type. If `T1` and `T2` are the same, then the caller ultimately decides
the index, but if `T1` and `T2` are different, then `Variant` needs to
decide the index.

Providing `Make1` and `Make2` works, since `Make1` only accepts a `T1`
argument while `Make2` only accepts a `T2` argument. Thus, if they are
the same, the caller disambiguates by calling one of the methods and
there is no compilation issue. If they are different, calling one of
them is the equivalent of calling one of the constructors (there is no
way to call `Make1` with a `T2` argument).

### Equality

Now the only remaining bit is overriding `Equals`, as we want two
variants containing equivalent values to be equivalent. In other words,
given another object, we would consider it equivalent if it has the same
type as this object, has the value *at the same index*, and the value of
the other object is equivalent to the value of this object:

``` c#
public override bool Equals(object obj)
{
    if (obj == null || !(obj is Variant<T1, T2>))
        return false;

    var other = (Variant<T1, T2>)obj;

    return Index == other.Index && Get().Equals(other.Get());
}
```

Overriding `Equals` usually means overriding `GetHashCode` in such a way
that equivalent objects hash to the same value. In our case, we can rely
on the value stored in the variant to implement this by simply
delegating hashing to it:

``` c#
public override int GetHashCode()
    => Get().GetHashCode();
```

### Multiple generic arguments

We have an implementation for a `Variant<T1, T2>` but we are looking at
providing variants from `Variant<T1>` all the way to
`Variant<T1, T2, T3, T4, T5, T6, T7, T8>`.

First, let's look at what would be common to all of these. The API
(`Is<T>`, `Get<T>` etc.) is implemented in terms of `IVariantHolder`.
Let's extract this into a base class. Since we are going to make all our
variants derive from it, it must be `public`, but we probably don't want
clients to derive from it as it is an implementation detail, so we will
provide an `internal` constructor. This will make this class
instantiable only within the assembly declaring it:

``` c#
public abstract class VariantBase
{
    private readonly IVariantHolder variant;

    public byte Index { get; private set; }

    public bool Is<T>() => variant.Is<T>();

    public T Get<T>() => ((VariantHolder<T>)variant).Item;

    public object Get() => variant.Get();

    internal VariantBase(IVariantHolder item, byte index)
    {
        variant = item;
        Index = index;
    }
}
```

Our `Variant<T1, T2>` ends up containing only the constructors, casts,
and equality:

``` c#
public sealed class Variant<T1, T2> : VariantBase
{
    // Calls base constructor
    private Variant(IVariantHolder item, byte index)
        : base(item, index)
    { }

    // T1 constructor, casts
    public Variant(T1 item)
        : this(new VariantHolder<T1>(item), 0)
    { }

    public static implicit operator Variant<T1, T2>(T1 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T1(Variant<T1, T2> item)
        => item.Get<T1>();

    public static Variant<T1, T2> Make1(T1 item)
        => new Variant<T1, T2>(new VariantHolder<T1>(item), 0);

    // T2 constructor, casts
    public Variant(T2 item)
        : this(new VariantHolder<T2>(item), 1)
    { }

    public static implicit operator Variant<T1, T2>(T2 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T2(Variant<T1, T2> item)
        => item.Get<T2>();

    public static Variant<T1, T2> Make2(T2 item)
        => new Variant<T1, T2>(new VariantHolder<T2>(item), 1);

    // Equality
    public override bool Equals(object obj)
    {
        if (obj == null || !(obj is Variant<T1, T2>)) return false;

        var other = (Variant<T1, T2>)obj;

        return Index == other.Index && Get().Equals(other.Get());
    }

    public override int GetHashCode()
        => Get().GetHashCode();
}
```

We can even hoist `Equals` to our base class, since we can replace the
`is` check `!(obj is Variant<T1, T2>)` with
`GetType() != obj.GetType()`:

``` c#
public abstract class VariantBase
{
    private readonly IVariantHolder variant;

    public byte Index { get; private set; }

    public bool Is<T>() => variant.Is<T>();

    public T Get<T>() => ((VariantHolder<T>)variant).Item;

    public object Get() => variant.Get();

    internal VariantBase(IVariantHolder item, byte index)
    {
        variant = item;
        Index = index;
    }

    public override bool Equals(object obj)
    {
        if (obj == null || GetType() != obj.GetType())
            return false;

        var other = (VariantBase)obj;

        return Index == other.Index && Get().Equals(other.Get());
    }

    public override int GetHashCode()
        => Get().GetHashCode();
}
```

Now our `Variant<T1, T2>` contains only constructors and casts:

``` c#
public sealed class Variant<T1, T2> : VariantBase
{
    // Calls base constructor
    private Variant(IVariantHolder item, byte index)
        : base(item, index)
    { }

    // T1 constructor, casts
    public Variant(T1 item)
        : this(new VariantHolder<T1>(item), 0)
    { }

    public static implicit operator Variant<T1, T2>(T1 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T1(Variant<T1, T2> item)
        => item.Get<T1>();

    public static Variant<T1, T2> Make1(T1 item)
        => new Variant<T1, T2>(new VariantHolder<T1>(item), 0);

    // T2 constructor, casts
    public Variant(T2 item)
        : this(new VariantHolder<T2>(item), 1)
    { }

    public static implicit operator Variant<T1, T2>(T2 item)
        => new Variant<T1, T2>(item);

    public static explicit operator T2(Variant<T1, T2> item)
        => item.Get<T2>();

    public static Variant<T1, T2> Make2(T2 item)
        => new Variant<T1, T2>(new VariantHolder<T2>(item), 1);
}
```

While C# doesn't provide a way to implement variable number of generic
arguments, constructors and casts for all types are identical, so we can
use a [T4 text
template](https://docs.microsoft.com/en-us/visualstudio/modeling/design-time-code-generation-by-using-t4-text-templates)
to generate all this code. Our template would iterate for each type and
emit the C# code for these:

``` text
<#
for (int types = 1; types <= 8; types++)
{
    // Comma-delimited string of types (eg. "T1, T2, T3")
    var args = String.Join(", ",
        Enumerable.Range(1, types).Select(i => "T" + i));

    // Type we are generating code for (eg. "Variant<T1, T2, T3>")
    var type = $"Variant<{args}>";
##>

    public sealed class <#= type #> : VariantBase
    {
        private Variant(IVariantHolder item, byte index)
            : base(item, index)
        {}

<#
    // For each type argument T1, T2, T3 etc.
    for (int i = 1; i <= types; i++)
    {
##>
        public Variant(T<#= i #> item)
            : base(new VariantHolder<T<#= i #>>(item), <#= i - 1 #>)
        {}

        public static implicit operator <#= type #>(T<#= i #> item)
            => new <#= type #>(item);

        public static explicit operator T<#= i #>(<#= type #> variant)
            => variant.Get<T<#= i #>>();

        public static <#= type #> Make<#= i #>(T<#= i #> item)
            => new <#= type #>(new VariantHolder<T<#= i #>>(item), <#= i - 1 #>);
<#
    }
##>
    }

<#
}
##>
```

I will not cover T4 templates in this blog post, just highlight that the
template above does generate all the `Variant<>` variations with the
appropriate constructors and cats.

I am currently working on a type library which includes this variant and
some other useful types: <https://github.com/vladris/Maki>.

## Summary

In this post we implemented a generic variant type in C#, going over:

* Variant types and how they are useful.
* Requirements for a variant type.
* Implementation:
  * API.
  * Type erasure.
  * Disambiguating between similar generic arguments.
  * Overrides for `Equals` and `GetHashCode`.
  * Implementations for various numbers of generic arguments.

[^1]: See ["tagged union" on
    Wikipedia](https://en.wikipedia.org/wiki/Tagged_union)
