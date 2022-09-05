# (Ab)using Maps

## Mapping pattern

Using hash maps (or dictionaries, or lookups) is a very natural way of
coding in some languages, especially dynamic languages, where usually an
object can be treated as a map itself, to which attributes and methods
can be added or removed at runtime.

In practice though, maps are often used to convert a value of one type
into a value of a different type. It is not uncommon to have very small
maps like

``` c++
const std::unordered_map<Foo, Bar> fooBarMap = {
    { foo1, bar1 },
    { foo2, bar2 },
    { foo3, bar3 }
};

...

auto bar = fooBarMap[foo];
```

Here it is useful to make a distinction between the pattern and the data
structure. The coding pattern itself is great - mapping a value from a
type to a value of another type should definitely be declarative. Below
is a counterexample of non-declarative mapping:

``` c++
Bar barFromFoo(const Foo& foo)
{
    if (foo == foo1)
        return bar1;
    if (foo == foo2)
        return bar2;
    if (foo == foo3)
        return bar3;
    ...
}
```

This is really ugly. As I mentioned in
[Clean Code - Part 1](https://vladris.com/blog/2016/01/04/clean-code-part-1.html),
branching should be avoided whenever possible, and this is
a good opportunity to use a declarative approach as opposed to a bunch
of branching logic. That being said, while the mapping pattern is great,
in C++ the data structure most developers default to is not the optimal
one for this.

## The problem with unordered_map

If you are coding in C++, odds are you care a little bit about the
runtime footprint of your code. In that case, you might be surprised to
learn that, while an `unordered_map` in C++ (or a lookup or hash map or
dictionary in any other language) has an average lookup cost of `O(1)`,
there are better ways to implement the above pattern.

A map in C++ is implemented as a red-black tree containing buckets of
hashed values. Calling `at()` on a map implies the given key has to be
hashed and the tree traversed to find the value. Calling `[]` on an
inexistent key will add it to the data structure, which might trigger a
rebalancing of the tree. There is a lot of work happening under the
hood, and while this makes sense for an `unordered_map` of arbitrarily
large size, for small lookups it is a lot of overhead.

## Alternatives

An alternative to `unordered_map` provided by the boost library is
`flat_map`[^1]. This has similar semantics to an `unordered_map`, but
the key-values are stored in a contiguous data structure so traversing
it is more efficient than walking a tree.

In general, there are a couple of approaches for keeping a hash map in a
linear data structure:

* The keys can be kept sorted, which has `O(N)` worst case insertion
  since it might require all elements to be moved to fit a new one and
  `O(logN)` lookup (binary search)
* The keys can be kept unsorted, which has `O(1)` insertion (simple
  append) but `O(N)` lookup (linear search)

For very small-sized lookups, the cost of hashing itself might
out-weight a linear traversal, so for a small N

``` c++
const std::unordered_map<Foo, Bar> fooBarMap = {
    { foo1, bar1 },
    { foo2, bar2 },
    { foo3, bar3 }
};

...

auto bar = fooBarMap[foo];
```

performs worse than

``` c++
const std::vector<std::pair<Foo, Bar>> fooBarMap = {{
    { foo1, bar1 },
    { foo2, bar2 },
    { foo3, bar3 }
}};

...

auto bar = std::find_if(
    fooBarMap.cbegin(),
    fooBarMap.cend(),
    [](const auto& elem) { return elem == foo; });
```

On my machine (using MSVC 2015 STL implementation), for an N of 5,
`find_if` on a vector is about twice as fast as the equivalent
`unordered_map` lookup.

## Initialization cost

There's event more hidden cost: `std::vector` manages a dynamic array
which is allocated on the heap. Having an `std::vector` initialized with
key-values as described above, even if more efficent than an
`unordered_map`, still has some associated cost in terms of heap
allocations (albeit smaller than `unordered_map`). `std::array` is a
much better suited container for cases when the key-values are known at
compile time, as `std::array` simply wraps a regular array which is not
allocated on the heap. So a more efficient (in terms of initialization
cost) way of declaring such a look up is

``` c++
const std::arrray<std::pair<Foo, Bar>, 3> = {{
    { foo1, bar1 },
    { foo2, bar2 },
    { foo3, bar3 }
}};
```

We can still apply the `std::find_if` algorithm on this array, but we
skip a heap allocation. Depending on the template types used, we might
be able to skip any allocations whatsoever (if both types are
trivial[^2]). For example, note that `std::string`, similarly to a
vector, wraps a heap-allocated `char*` and constructing it requires heap
allocations. `const char*` to a string literal on the other hand is just
a pointer to the `.rodata` segment. So this

``` c++
const std::array<std::pair<std::string, Bar>, 3> = {{
    { "foo1", bar1 },
    { "foo2", bar2 },
    { "foo3", bar3 }
}};
```

performs three heap allocations (for `"foo1"`, `"foo2"`, and `"foo3"`),
while the (mostly) equivalent

``` c++
const std::array<std::pair<const char*, Bar>, 3> = {{
    { "foo1", bar1 },
    { "foo2", bar2 },
    { "foo3", bar3 }
}};
```

shouldn't perform any allocations.

## associative_array

Since in practice maps are often used to implement the above described
pattern of mapping a value from one type to a value of a different type
for a small set of known values, it would be great to combine the
efficiency of an array with the nice lookup semantics of an
`unordered_map` conatiner.

I propose a generic container of the following shape:

``` c++
template <
    typename TKey,
    typename T,
    size_t N,
    typename KeyEqual = key_equal<TKey>>
struct associative_array
{
    std::pair<TKey, T> m_array[N];
    ...
};
```

`keq_equal` should simply resolve to `==` for most types, but be
specialized for strings types (to use `strcmp`, `wcscmp` etc.) and allow
clients to specialize their own `key_equal` when needed.

``` c++
template <typename T> struct key_equal
{
    bool operator(const T& lhs, const T& rhs)
    {
        return lhs == rhs;
    }
};

template <> struct key_equal<char*>
{
    bool operator(const T& lhs, const T& rhs)
    {
        return strcmp(lhs, rhs) == 0;
    }
};
...
// specializations for wchar_t and const variations of the above
```

Satisfying the container concept is fairly easy (eg. `size()` would
return `N`, iterators over the member array are trivial to implement
etc.), the only interesting methods are `find()`, `at()`, and
`operator[]`:

``` c++
...
struct associative_array
{
    ...
    iterator find(const TKey& key)
    {
        return std::find_if(
            begin(),
            end(),
            [&](const auto& elem) { return KeyEqual{}(key, elem.first); });
    }

    T& at(const TKey& key)
    {
        auto it = find(key);
        if (it == end())
            throw std::out_of_range("...");
        return it->second;
    }

    T& operator[](const TKey& key)
    {
        return find(key)->second;
    }
    ...
};
```

`find()` wraps `std::find_if` leveraging `KeyEqual` (with default
implementation as `key_equal`), `at()` wraps a bounds-checked `find`,
while `operator[]` does not check bounds. `const` implementations of the
above are also needed (identical except returning `const T&`).

Such a container would have similar semantics to `std::unordered_map`
(minus the ability to add elements given a key not already present in
the container) and the same performance profile of `std::array`:

``` c++
const std::associative_array<Foo, Bar, 3> fooBarMap = {{
    { foo1, bar1 },
    { foo2, bar2 },
    { foo3, bar3 }
}};

...

auto bar = fooBarMap[foo];
```

Note the only syntax difference between above and `unordered_map` is the
container type, the extra size `N` which needs to be specified at
declaration time, and an extra pair of curly braces. In practice, this
should have a significantly better lookup time than an unordered_map for
a small N (linear time, but since N is small and no hashing or heap
traversal occurs, should clock better than a map lookup) and virtually
zero initialization time - depending on the `TKey` and `T` types used,
it is possible to declare an `associative_array` as a `constexpr` fully
evaluated at compile-time.

[^1]: Boost `flat_map` documentation is
    [here](http://www.boost.org/doc/libs/1_56_0/doc/html/boost/container/flat_map.html).

[^2]: For more details on trivial types, see the [is_trivial type
    trait](http://www.cplusplus.com/reference/type_traits/is_trivial/).
