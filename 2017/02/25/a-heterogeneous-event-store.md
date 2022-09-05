# A Heterogeneous Event Store

I recently stumbled upon a heterogeneous event collection which turned
out to pose an interesting design problem. We are using library code (we
can't change) that provides a templated `Event` to which we can
register callbacks and which we can raise later to invoke the callbacks.
The interface looks like this:

``` c++
/* Library code */
template <typename T> struct Event
{
    // T is a callable object, like std::function
    void Register(T&& callback) { /* Register a callback */ }

    // Raises the event and forwards the arguments to the callbacks
    template <typename ... Args>
    void Raise(Args&& ... args) { /* Invoke all registered callbacks */ }
};
```

A stub implementation that validates client code is typed properly would
be:

``` c++
template <typename T> struct Event
{
    // T is a callable object, like std::function
    void Register(T&& callback)
    {
        /* Register a callback */
        _callback = std::move(callback);
    }

    // Raises the event and forwards the arguments to the callbacks
    template <typename ... Args>
    void Raise(Args&& ... args)
    {
        /* Invoke all registered callbacks */
        _callback(std::forward<Args>(args)...);
    }

    T _callback;
};
```

Note that unlike the real implementation, this only stores the last
registered event, but that is irrelevant for the purpose of this post.
I'm providing the code just to have something to compile against
(otherwise `Raise` would happily swallow any combination of arguments
passed to it). In reality, a more complex implementation would maintain
a list of callbacks, but this is sufficient for framing the design
problem.

The event collection which was wrapping a set of library events looked
like this:

``` c++
using LaunchCallback = std::function<void()>;
using SaveCallback = std::function<void(const std::string&)>;
using ExitCallback = std::function<void()>;

class EventStore
{
public:
    void OnLaunch(LaunchCallback&& callback)
    {
        m_launchEvent.Register(std::forward<LaunchCallback>(callback));
    }

    void OnSave(SaveCallback&& callback)
    {
        m_saveEvent.Register(std::forward<SaveCallback>(callback));
    }

    void OnExit(ExitCallback&& callback)
    {
        m_exitEvent.Register(std::forward<ExitCallback>(callback));
    }

    void RaiseLaunch()
    {
        m_launchEvent.Raise();
    }

    void RaiseSave(const std::string& fileName)
    {
        m_saveEvent.Raise(fileName);
    }

    void RaiseExit()
    {
        m_exitEvent.Raise();
    }

private:
    Event<LaunchCallback> m_launchEvent;
    Event<SaveCallback> m_saveEvent;
    Event<ExitCallback> m_exitEvent;
};
```

Sample usage of the `EventStore`:

``` c++
EventStore store;

// Register a couple of callbacks
store.OnLaunch([]() { /* Do stuff */ });
store.OnSave([](const auto& arg) { /* Do stuff */ });

// Raise an event
store.RaiseSave("Some file name");
```

Looking at `EventStore`, it's obvious that there is a lot of repetition
involved: hooking up a new event involves aliasing a new callback,
adding a new member to the class, and adding the corresponding
registration and `Raise` member functions which end up being copy/pastes
of the other ones. There must be a better way!

An initial idea would be to use some sort of associative container
(hopefully something [better than an
unordered_map](http://vladris.com/blog/2016/04/24/abusing-maps.html)),
but there is an interesting complication due to the fact that some of
the various events are actually of different types.
`Event<std::function<void()>>` has a different type than
`Event<std::function<void(const string&)>>`. There are potential
workarounds to explore, like standardizing on a single type and
requiring clients to, for example, only use callbacks that do not take
any arguments. Another option would be to pass in some base object to
each event and let each callback re-interpret it. This takes us down the
wrong path though. We don't need to do any runtime lookup - the initial
code doesn't.

From the repetition in `EventStore`, it should become apparent that we
need some form of templated `Register` and `Raise` that would work for
each type of event we care about. A quick sketch of our function
signatures should look something like this:

``` c++
template </* ??? */>
void Register(/* ??? */)
{
    // Register callback to the appropriate event
}

template </* ??? */, typename ... Args>
void Raise(Args&& ... args)
{
    // Raise the appropriate event, forwarding args to it
}
```

It is also clear that we need a way to store all of the events we need
in our class. Since they are of heterogeneous types, we can't store
them in a map or equivalent, but we don't need to. `std::tuple` was
build exactly for this:

``` c++
std::tuple<Event<LaunchCallback>,
           Event<SaveCallback>,
           Event<ExitCallback>> m_events;
```

Now the only remaining question is how to templatize our two member
functions to enable a lookup in the tuple. One approach would be to use
an enum:

``` c++
enum class EventType : size_t
{
    LaunchEvent = 0,
    SaveEvent,
    ExitEvent,
};
```

Given this enum, we can templatize on its values:

``` c++
class EventStore
{
public:
    template <EventType eventType, typename T>
    void Register(T&& callback)
    {
        std::get<static_cast<size_t>(eventType)>(m_events).Register(std::forward<T>(callback));
    }

    template <EventType eventType, typename ... Args>
    void Raise(Args&& ... args)
    {
        std::get<static_cast<size_t>(eventType)>(m_events).Raise(std::forward<Args>(args)...);
    }

private:
    std::tuple<Event<LaunchCallback>,
               Event<SaveCallback>,
               Event<ExitCallback>> m_events;
};
```

Callers can use this new implementation like this:

``` c++
EventStore store;

// Register a couple of callbacks
store.Register<EventType::LaunchEvent>([]() { /* Do stuff */ });
store.Register<EventType::SaveEvent>([](const auto& arg) { /* Do stuff */ });

// Raise an event
store.Raise<EventType::SaveEvent>("Some file name");
```

With this implementation, we preserve the ability to have polymorphic
events but only need to implement the `Register` and `Raise` functions.
Adding a new event type now only requires aliasing the callback, adding
an enum member, and extending our member tuple by adding the new `Event`
to it.

The only drawback of this approach is the fact that we need to manually
keep the enum and the tuple in sync. This is not too bad, because if we
try to call `std::get` with a number higher than the size of the tuple,
we get a compile time error. If we accidentally swap two events, if they
are of incompatible types (for example `Event<LaunchCallback>` and
`Event<SaveCallback>`, as one expects callbacks of type
`std::function<void()>` and the other expects
`std::function<void(const std::string&)>`), we get a compile-time error
because `Register` and `Raise` calls would fail to compile (attempting
to pass in callback/arguments of incompatible types). If we accidentally
swap two events of the same type, (`Event<LaunchCallback>` and
`Event<ExitCallback>`, since both `LaunchCallback` and `ExitCallback`
are aliased to the same `std::function<void()>`), runtime behavior is
equivalent, it just makes reading the code confusing. Now we end up
storing launch callbacks inside what we called `Event<ExitCallback>` and
vice-versa. Runtime is not affected, as we would also raise
`Event<ExitCallback>` by calling `Raise<EventType::LaunchEvent>`, but
it's not ideal. We could drop the aliases altogether and simply have:

``` c++
std::tuple<Event<std::function<void()>>,
           Event<std::function<void(const string&)>>,
           Event<std::function<void()>> m_events;
```

This solves the above issues but is not very readable. There are other
options, like picking different names for the aliases - instead of
naming the event, have them name the type of callback. Either way,
effectively what we are doing is a mapping from an enum into a set of
`Event` types. We can actually push more information to the type system
and get rid of the need to do this mapping. We do that by making sure
our events are always of different types, even if the callback
signatures are the same. One way of achieving this is wrapping `Event`
and defining different types for each of our events:

``` c++
template <typename T> struct EventWrapper
{
    Event<std::function<T>> m_event;
};

struct LaunchEvent : EventWrapper<void()> { };
struct SaveEvent : EventWrapper<void(const std::string&)> { };
struct ExitEvent : EventWrapper<void()> { };
```

Note this is type information used only by the compiler and doesn't
bring any runtime overhead to our code. Inheritance is used here just so
we don't have to repeat declaring `m_event`, we could have just as well
declared each struct independently. Now we can update the member tuple
to store an event of each of these types:

``` c++
std::tuple<LaunchEvent, SaveEvent, ExitEvent> m_events;
```

Since they are of different types, we no longer need an enum to index
into the tuple, we can do it by type (note `std::get` indexed by type
requires that the tuple contains distinct types, which is not the case
for `Event<LaunchCallback>` and `Event<ExitCallback>`, but it is for
`LaunchEvent` and `ExitEvent`):

``` c++
template <typename T, typename Callback>
void Register(Callback&& callback)
{
    std::get<T>(m_events).m_event.Register(std::forward<Callback>(callback));
}

template <typename T, typename ... Args>
void Raise(Args&& ... args)
{
    std::get<T>(m_events).m_event.Raise(std::forward<Args>(args)...);
}
```

The `Callback` template argument in `Register` can be deduced once `T`
is specified. The full implementation is:

``` c++
template <typename T> struct EventWrapper
{
    Event<std::function<T>> m_event;
};

struct LaunchEvent : EventWrapper<void()> { };
struct SaveEvent : EventWrapper<void(const std::string&)> { };
struct ExitEvent : EventWrapper<void()> { };

class EventStore
{
public:
    template <typename T, typename Callback>
    void Register(Callback&& callback)
    {
        std::get<T>(m_events).m_event.Register(std::forward<Callback>(callback));
    }

    template <typename T, typename ... Args>
    void Raise(Args&& ... args)
    {
        std::get<T>(m_events).m_event.Raise(std::forward<Args>(args)...);
    }

private:
    std::tuple<LaunchEvent, SaveEvent, ExitEvent> m_events;
};
```

Callers can use it like this:

``` c++
EventStore store;

// Register a couple of callbacks
store.Register<LaunchEvent>([]() { /* Do stuff */ });
store.Register<SaveEvent>([](const auto& arg) { /* Do stuff */ });

// Raise an event
store.Raise<SaveEvent>("Some file name");
```

In this case, adding a new event requires declaring a new struct and
adding it to the tuple. Since we are retrieving the event by its type,
no mapping is involved.
