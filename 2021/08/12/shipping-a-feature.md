# Shipping a Feature

I recently did a small presentation for my team talking about what it
takes to ship a feature to customers in a product like Office. This
includes much more than getting the functionality implemented and
tested. In fact, figuring out the design and implementing it accounts
for about 20% of the work (from prototype to implementation, including
tests). The remaining 80% is taking the feature from *functional* to
*shippable*. In this post, I'll go over some of these non-functional
aspects of shipping code to customers.

Most of the aspects I will talk about rely heavily on telemetry. This is
a consequence of today's connected world. Software is shipped over the
internet every month or every week (or every day), as opposed to every
year or every other year. Connectivity also enables us to get signals on
how the software is performing and adjust accordingly.

## Reliability

Reliability can make or break a feature. It is the bedrock of quality.
If the feature is unreliable and causes crashes or data corruption,
nobody will use it. The most egregious of reliability issues is data
loss - for example, if a user writes a paragraph and the application
crashes before this paragraph gets saved anywhere, forcing the user to
re-type it causes a huge loss of trust.

In terms of software engineering, beyond comprehensive testing, we need
to have telemetry in place to report errors. There are several ways to
quantify reliability. For example, a couple of common measures are the
*ICE* and *ACE* scores.

*ICE* stands for *Ideal Customer Experience*. This measures the
reliability of a scenario including all errors encountered during a
user's session. 100% means no session encountered any errors.

*ACE* stands for *Adjusted Customer Experience*. This is similar to ICE,
except it only measures unexpected errors. In a complex system, some
errors are expected, and the software can recover from them without
impacting the user experience. For example, a background call fails but
succeeds on retry.

Both measures are important - one measures the overall reliability of
the system, the other measures the user experience impact.

100% ICE and ACE scores are ideal, but hard to achieve in a complex
system. That said, shipping a feature should include setting a target
ICE and ACE score (for example 99%), making sure telemetry is there to
provide the signal, and working towards that goal as the feature gets
exposed to more and more users (more on exposure below).

## Performance

Performance is another fundamental aspect of any feature. Of course, the
first rule of performance tuning is to measure. Shipping a feature
includes instrumentation to measure various scenarios like time to load,
time to render etc. Customers run the code on different types of
hardware, and it is important to understand what the actual user
experience is (in the 95th percentile).

Like reliability, we should set some goals, look at telemetry, and keep
improving. Important to note that this is not a one-time thing, as
software gets updated and functionality is added, performance can easily
degrade. Performance numbers should show up on a dashboard and should be
reviewed periodically.

## Security

Microsoft created the [Security Development
Lifecycle](https://www.microsoft.com/en-us/securityengineering/sdl/) for
building better, safer, software internally. In 2008, this was shared
with the rest of the world and it continues evolving. I won't cover all
security best practices as it would take way more than a blog post, but
I will touch on a few points:

* [Threat
  modeling](https://www.microsoft.com/en-us/securityengineering/sdl/threatmodeling)
  \- non-trivial features should have a threat model - a data flow
  diagram which captures the various entities in the system and the
  trust boundaries. The Microsoft Threat Modeling Tool can analyze a
  threat model and list out potential attacks and suggested
  mitigations.

* Adhering to standards and best practices - for example always
  connecting over HTTPS instead of HTTP, using cryptographically
  strong algorithms and so on.

* Leveraging available tools to ensure a feature is secure. For
  example, running static code analysis to identify potential attack
  vectors and [fuzzing](https://en.wikipedia.org/wiki/Fuzzing) inputs.

The bar for security should be very high, and a feature should never
roll out to customers before ensuring it is secure.

## Privacy

Customer privacy is extremely important. If our feature handles any
customer data (data about the customer) or customer-owned data (data
generated by the customer), we need to be very careful how we process
it, store it, and who can access it. I talked extensively about data
handling in my previous blog post, [Changing Data Classification Through
Processing](https://vladris.com/blog/2020/11/27/changing-data-classification-through-processing.html).

## Compliance

Compliance ensures we don't open ourselves up to liability. An example
of this is third party notices for open-source libraries - many
open-source licenses require you to mention you are using the library.
Another example is GDPR compliance - EU citizens can request to view all
the data a company has on them, or request to be "forgotten" (have
their data removed from the system).

Compliance also means our software adheres to various standards some
customers might require. An example of this is data sovereignty:
countries like Germany and China require data to be stored in data
centers within their country. Another example is ensuring our software
meets the requirements for certain certifications like SOC2, without
which certain organizations wouldn't be able to use it.

Our feature needs to be compliant before shipping it to customers.

## Accessibility

The Microsoft mission statement reads as follows:

> *Our mission is to empower every person and every organization on the
> planet to achieve more.*

Making our software accessible to people with disabilities is very
important. Here are a few aspects we need to consider when shipping
features users interact with:

* Contrast ratio - the contrast between foreground colors and
  background colors needs to be good enough so everything is legible
  for people with vision impairment.
* High contrast - operating systems provide high-contrast modes for
  the visually impaired. Features need to be tested in high-contrast
  mode to ensure they are still usable.
* Screen reader support - UI elements need to be annotated so screen
  reader software can describe them correctly.
* Keyboarding - some users navigate exclusively using keyboard
  shortcuts, so a feature needs to implement common keyboard shortcuts.
* Focus - a feature needs to properly handle focus switching. This
  includes tab order, making sure focus doesn't get trapped on a
  single control or subset of the controls etc.
* Touch - touchscreen devices without keyboard usually have larger
  touch targets. We need to make sure our feature works well on
  touchscreens too.

## World readiness

World readiness means our feature can ship in markets across the world,
and usually entails two different aspects: *globalization* and
*localization*.

Globalization means our feature works in different cultures, with
different (OS-level) culture settings, without requiring additional,
culture-specific changes. An example is the date format. In US, the date
format is `mm-dd-yyyy`, while in Europe it is `dd-mm-yyyy`. Japan uses
`yyyy-mm-dd`. When displaying dates, we should format them using the
current culture's format, not assume and hardcode any specific format.

Another important aspect of globalization is to ensure the symbols used
as icons make sense for everyone. For example, using a starry wizard hat
to represent a setup wizard works in Western cultures but might not make
sense all around the world.

Localization deals with translating UI strings into all the languages in
which the feature ships. From a developer perspective, UI layout is
important here: in some languages, words are on average longer than in
English, so text might overflow its boundaries when translated. For
example, "Add job" in English becomes "Auftrag hinzufügen" in
German.

We also have right-to-left languages like Arabic and Hebrew. We need to
ensure our layout works as expected in such languages.

## Feature gating

In a mature product like Office, making changes feels very much like
rebuilding an airplane in flight: we can add and modify features, but
breaking anything is catastrophic. One best practice of shipping
features in such conditions is using feature gates. A feature gate is a
toggle that determines whether a code path should be exercised or not.
This allows us to release a build containing a half-developed feature.
If it is not quite ready to see the light of day, the feature gate is
closed, and the code never runs in production.

Once the feature is ready, we can toggle the feature gate and start
running the code. In case we notice any issues, we can flip the feature
gate back off and mitigate customer impact while the issue gets
resolved.

Once the feature is mature and has been exercised by customers for a
while, we can go back and clean up the feature gate.

## Flighting

Feature gates are a simple on/off switch. Flighting changes is a more
mature version of progressive exposure. We usually have a set of rings
through which we release. For example, we could have a dogfood ring, for
the team to try out the code themselves, before shipping further; we
could have a beta tester ring, for customers who sign up to get preview
features; finally, we could have a production ring, containing all other
customers.

We should be able to progress a feature through these different rings,
and within the rings, only expose a percentage of customers to the
feature.

Flighting infrastructure needs to exist for this. From a feature
development perspective, we need a roll out plan defining what telemetry
signals we want to see to be comfortable increasing exposure of our
feature.

## Experimentation

Finally, we need a way to measure whether a feature is successful or not
and determine whether iterating on it improves things or makes them
worse. First, we need to define what success means - what metrics are
most important for our product. Once we have these, we can run an A/B
test when introducing a new feature or iterating on an existing feature.
We have a control group seeing the old behavior and a treatment group
seeing the new behavior. We can then look at the key metrics we defined
and see how they look for both control and treatment. Did the new code
move the needle?

## Summary

Shipping a feature takes a lot of work beyond the initial functional
implementation. In this post we looked at some of these various aspects:

* Reliability
* Performance
* Security
* Privacy
* Compliance
* Accessibility
* World readiness
* Feature gating
* Flighting
* Experimentation

All of these need to be taken into account when we ship code to our
customers.