# Time and Complexity

2020 being a leap year, it's a good opportunity to talk about how we
track time. We'll start with that, but this post is as a reflection on
the inherent complexity of the physical world and human societies.

There is a famous blog post, [Falsehoods programmers believe about
time](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time),
which covers some well known pitfalls like *years have 365 days* or
*February is always 28 days long*. Unfortunately, a lot of software has
such assumptions hardcoded and things go awry.

But before talking about software, let's step back and look at how we
measure time.

## Atomic Clocks

[Atomic clocks](https://en.wikipedia.org/wiki/Atomic_clock) provide an
extremely precise measure of the passage of time. These clock don't
gain or lose a single second over hundreds of millions of years. These
devices are beautiful: they can provide a monotonic count of the passage
of time with an incredible precision.

In fact, we have the [International Atomic
Time](https://en.wikipedia.org/wiki/International_Atomic_Time) standard,
or TAI, which is defined, according to Wikipedia, by the weighted
average of 400 atomic clocks from over 50 laboratories across the world.
This is an extremely precise measure of time passing on Earth.

It is also not practical enough to be used as the basis of our
calendars. Even with a very accurate, high-resolution, atomic clock, we
still need to account for the fact that the Earth orbits around the Sun
(so we get seasons), and spins around its axis (so we get day and
night). Let's see why there isn't a simple mathematical function from
atomic clock tick to year-month-day-hour-minute-second.

## Leap Years

Leap years were introduced to account for the fact that Earth's orbit
around the Sun is slightly longer than exactly 365 days. Without
adjusting for this fact, the seasons would gradually shift around the
calendar. But a leap year doesn't necessarily occur every 4 years.
Turns out Earth's orbit is slightly smaller than 365 days and 6 hours,
so adding one day every 4 years would cause seasons to drift the
opposite direction. The leap year rule is actually

> A leap year is every year divisible by 4, except for years divisible
> by 100, unless they are also divisible by 400.

So years like 2020, 2024, 2028 and so on are leap years. But years like
1700, 1800, 1900, are *not* leap years, because they are divisible by
100. Except 1600, and 2000, which are not only divisible by 100, but
also by 400.

We started with a simple model of a precise, monotonic atomic clock
measuring ticks, but when get to user-friendly time, we end up with
complex business rules that aim to account for the physical world. But
it gets more complicated.

## Leap Seconds and Standards

If leap years were all there is to it, we could've easily mapped an
atomic clock tick to a precise date time value. But it gets more
complicated. Turns out Earth's rotation is not constant - it is
irregular, and trends towards slow down. Major earthquakes can affect
the momentum of the rotation. Tidal interaction with the moon is also
slowing down the speed of rotation over millions of years.

[UT1](https://en.wikipedia.org/wiki/Universal_Time), the Universal Time
standard based on Earth's rotation, is drifting from the
[UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time), the
Coordinated Universal Time, which uses atomic clocks to measure time.
Because of this, UTC had to introduce leap seconds. A leap second aims
to bring UTC time (based on atomic clock measurements) back in sync with
UT1 time (time as observed astronomically), so they are not more than 1
second apart.

A leap second adds one second to a day, so we end up with a 61
seconds-long minute. Leap seconds are usually added at the end of the
month, UTC time. For example, on June 30th 2015, the UTC time was, at
some point, 23:59:60. This effectively makes a day 1 second longer.

There is no formula for this: as we measure time both astronomically and
atomically, a standards body decides when a leap second is introduced
and notifies the world every 6 months. In fact, the standard UTC time we
use is, as of the time of this writing, 37 seconds behind the TAI.

## Other Requirements

Besides the physical realities of measuring time atomically and
astronomically, we have multiple other requirements.

We have daylight saving time, which moves the clocks forward 1 hour in
spring and 1 hour backward in the fall. This creates a 23 hour-long day
in the spring and a 25 hour-long day in the fall (falsehood programmers
believe about time: *all days have 24 hours*). This is also not standard
across the world: some countries observe daylight saving while others
don't.

We have time zones, which don't neatly divide the earth in 24
equal-width slices, rather are set at geopolitical boundaries. Time
zones aren't even necessarily multiples of 1 hour: India is 5 hour and
30 minutes ahead of UTC.

Also note that daylight saving and time zones get updated: Russia
recently stopped observing daylight saving while China went from 5
different time zones to a single one, even though its geography hasn't
changed.

## Inherent Complexity

Even with an exact atomic clock, when taking into account year length
and day and night cycles, we have to introduce additional rules to
determine the date and time, like leap years and daylight saving. Not
only that, international standards bodies determine when leap seconds
occur, while countries are free to decide which time zone or time zones
they are using.

I believe this is typical of any non-trivial problem space we tackle
with software. When we try to model the physical world, things get
messy. They get messier with humans in the system: laws, standards, and
expectations introduce other arbitrary rules. Software needs to be
complex to handle the real world.

## Accidental Complexity

The above conclusion might seem to stand against pretty much everything
I wrote on this blog, where I try to argue for clean and simple code.
Why bother if any real world piece of software is destined to grow
complex? The reason is that there is enough complexity inherent in
dealing with the real world, without us having to introduce more. We
don't need to make things worse than they are. To quote a couple of
lines from [The Zen of
Python](https://www.python.org/dev/peps/pep-0020/):

> Simple is better than complex.
>
> Complex is better than complicated.

We try to keep things simple. Sometime simple is not enough, we need
complex solutions to complex problems. But at least let's not make them
complicated. A clean, well-crafted system, with rules properly
encapsulated can still be fairly easy to work with. If developers
introduce additional complexity which stems not from the problem domain
but from coding practices, then the ability to reason about and maintain
the system drops precipitously. This is called *accidental complexity*.

We should always ask ourselves whether the complexity we are dealing
with is inherent or accidental. The former is unavoidable, the latter
should be avoided at all cost.
