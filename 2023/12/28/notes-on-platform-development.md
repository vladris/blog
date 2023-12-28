# Notes on Platform Development

I spent the past few years building a platform for Loop components within the
Microsoft 365 ecosystem. While some of the learnings might only apply to our
particular scenario, I think some observations apply broadly.

We’ve been using 1P/2P/3P to mean our team (1P), other teams within Microsoft
(2P), and external developers (3P). Loop started with a set of 1P components and
we set out to extract a developer platform out of these that can be leveraged by
other teams. We currently have a set of 2P components built on our platform, and
a 3P developer story centered around [Adaptive Cards](https://learn.microsoft.com/en-us/microsoftteams/platform/m365-apps/cards-loop-component).

In this blog post I’ll cover some of my learnings with regard to platform
development.

## 1P != 3P

Aspirationally, we set out with the stated goal of *1P equals 3P*, meaning 3rd
party developers should be building on the same platform as 1st party
developers. Looking at it another way, if the platform is good enough for 1st
party, it should be just as good for 3rd party - this is a statement of platform
capabilities and maturity and a lofty goal.

That said, I don’t think this is realistic, especially within a product like
Office, where user experience is paramount. That is because we have two
audiences to consider: we have the developer audience - users building on our
platform, and we have Office users, people who get to use the end product.
Mediating between the two is quite a challenge.

A simple example is the classic performance/security tradeoff. Especially as
Loop components are embedded in other applications, what level of isolation do
we provide? Loop components are built with web technology. An iframe provides
great isolation (best security) but iframes add performance overhead (worse
perf). If we host a Loop component without an iframe, we get better performance,
but we open up the whole DOM to the component. If we threat model this, we
immediately see that we don’t necessarily need isolation for Loop components
developed within Microsoft (we don’t expect our partner teams to write malicious
code) but we absolutely need to isolate code written by 3rd party developers. Of
course, we could say “just isolate everything”, which might even have other
advantages, but do we want to take the perf hit? Our *other* audience, people
who use our product, would be negatively impacted by an overhead we can
technically avoid.

Another example in the same vein: overall user experience. The more we make Loop
components feel like part of the hosting app, the smoother the end user
experience is. On the other hand, we can’t realistically test every single Loop
component built by any 3rd party developer. The way Office services and products
are deployed and administered, tenant admins can configure which 3rd party
extensions are enabled within the tenant. The Microsoft tenant we use internally
has set some set of extensions available, but not all. That means there are
always 3rd party extensions we never even see. Now if one of these extensions
doesn’t work properly (errors out, looks out of place, is slow etc.), end users
might end up dissatisfied with the overall experience of using Office products.
For internally developed components, we get to dogfood and keep a high bar, but
this doesn’t scale to a wide developer audience. Our current approach is to
offer 3rd party development via Adaptive Cards. This way, we don’t run 3rd party
code on clients and we have a set of consistent UI controls. Ideally, we’d like
to enable custom code but this at the time of writing we’re still thinking
through the best approach considering all of the challenges listed above.

Finally, I think another key difference is the product goals. The platform
audience are the developers, but the product audience are the users. There’s
usually a tension between these. For example, an internal team builds a Loop
component. They come up with a requirement that is a “must” to deliver their
scenario. For example, we had a component developed by a partner team that asked
us to check the tenant’s Cloud Policy service to see whether the component
should be *on* or *off*. This makes perfect sense in this case, since the
backing service might not be running in the tenant. We offer tenant admins a
different way to control 3rd party extensions, so this platform capability would
not make sense for a 3rd party. In general, a lot of our internal platform
capability requests come from the desire to provide the best possible end user
experience. If our only customer were the developers using the platform, we
would probably say “no” to some of these - not general enough, doesn’t benefit
3rd party etc. But, of course, Office has way more users than developers.

I think the 1P/3P challenge is common to most platforms built from within
product teams (or supporting product teams within the same company). With Loop,
this is compounded by the fact we are deeply integrated within other
applications. I can think of some notable examples when the strong push for a
“1P equals 3P” platform ended up disastrously - Windows Longhorn was supposed to
be built on a version of .NET that was just not good enough for core OS pieces.
I can also think of many platforms that provide sufficient capabilities for 3rd
party developers but 1st/2nd party developers don’t use. And I think this is
OK - building a platform for 3P lets you focus on the developer community needs.
Supporting 1P/2P might be best served by focusing on the product goals and
unique scenario needs rather than trying to generalize to a public platform.

## Life stages

A platform goes through several life stages, each with its own characteristics
and challenges. Looking back at how our platform evolved (and how I foresee the
future), a successful platform goes through 4 life stages: *incubation*, *0 to
1*, *stabilization*, and *commoditization*.

### Incubation

At this stage, it’s all one team building both the what-will-become-a-platform
and the product supported by this platform. During the incubation stage, the
platform doesn’t really have any  users (meaning developers leveraging the
platform). We are free to toy with ideas. If we want to make a breaking change
to an API, we can easily do it and fix the handful of internal calls. At this
point, everything is in flux - the canvas is blank and we have plenty of room to
innovate.

On the other hand, we don’t really have a clear idea of what developers would
need out of the platform - we know what the main scenario we are supporting
needs, but we don’t have a feedback loop yet. At this stage, we need to rely on
experience and intuition to set some initial direction.

### 0 to 1

This is the biggest growth stage. “0 to 1” is a nod to Peter Thiel’s [Zero to One](https://www.goodreads.com/book/show/18050143-zero-to-one) book. The
platform goes from no users to a few users - and by “users” here I mean
developers. Taking the platform from 0 (or incubation) to 1, means supporting a
handful of “serious” production scenarios.

We now have a feedback loop and developers able to give us requirements - we can
now understand their needs rather than have to divine them ourselves. As a side
note, this is the approach we took with Loop, where we worked closely with a set
of 2P partners to light up scenarios and grow the platform to support these.

At this stage, it’s already difficult to make breaking changes. Since there are
already a set of dependencies on the platform, a breaking change requires a lot
of coordination. Or some form of backwards compatibility. Or legacy support.
There are different ways to go about this (maybe in another blog post), but the
key point is we can no longer churn as fast as we could during the incubation
stage. And added costs at the 0 to 1 stage are painful.

Another challenge is generalization. We have a handful of partners with a
handful of requests for the platform. And we’re in the growth stage, so we most
likely need to move fast. There’s a big tension between how fast we can light up
new platform capabilities and how much time we spend thinking through design
patterns and future-proofing. If we just say “yes” to every ask, we can move
fast but risk ending up with a very gnarly platform that has many one-off pieces
and a very inconsistent developer story. On the other hand, we can spend a lot
of time iterating on design and predicting how an incoming requirement would
scale when the platform is large, all the way until our partners give up on us
or funding runs out. There is no silver bullet for this - you always end up
somewhere in the middle, with parts of the platform that you wished were done
differently, but hopefully still alive and kicking in the next stage.

### Stabilization

At this point, enough developers depend on the platform that ad-hoc breaking
changes are no longer possible. By “stabilization” I don’t mean the platform
stops growing - in fact, this is the stage where we get most feedback and
requests. But while the platform continues to grow incrementally, changes become
even more difficult as they can break the whole ecosystem.

There are now enough user that early design decision that proved wrong become
obvious, but it’s too late to change them. This is a natural “if I knew then
what I know now” point for any platform that can’t really be avoided.

This is the point where most platform start producing new major version numbers
that aim to address large swats of issues and add new bundles of functionality.
But while during the incubation stage, a change could land in a few days, and in
the 0 to 1 stage maybe weeks or at most months, breaking changes at this stage
take years to land - many developers means not all of them are ready right-away
to update their code to the newest patterns. The platform needs some form of
long-term support for older versions and deprecation/removal becomes a long
journey.

On the other hand, the core of the platform is stable by now and battle-tested.
The final step is the platform becoming a commodity.

### Commoditization

At this stage, the platform is mature and robust. A large developer community
depends on it and the platform is mostly feature complete. Some new requirements
might pop up from time to time, but not very often.

At this stage developers rely on existing behaviors and change is next to
impossible. That’s because a lot of the developer solutions are also “done” by
now and people moved on. Nobody wants to go back and update things to support
API changes. The platform is a useful commodity.

This is also the stage where active development slows down and fewer engineers
are required to keep things going. We haven’t reached this stage with Loop, we
are still growing the platform and moving fast. But any successfully platform
should reach this stage - a low-churn state where its capabilities (and gotchas)
are well understood and reliable.

Each of the stages require a different approach to evolving the platform. The
speed with which we add capabilities, churn, how updates are rolled out, how we
design new features - all happen in different ways and at a different pace
depending on where the platform is and its number of users.

## Summary

In this post I covered two main aspects of platform development: the tension
between supporting 3rd party developers and ensuring end users have the best
possible experience; and the different stages of a platform. As usage increases,
changes become more difficult and early decisions solidify, for better or worse.

If I look at other platforms, I can easily see how they went through the same
growing pains and challenges.

I’ll probably have more to write on the topic of platform development, since
this has been my main job for a while now.
