---
layout: post
title: "Are You Really Ready to Start Adding Business Logic?"
date: 2018-09-27
tags: [process, design]
---

The interaction between business and technology teams, especially at startups, 
often drives us towards having the earliest possible deliverables. Acronyms like
MVP and POC are used heavily to explain that the emphasis in any given endeavor is to
get some basic version up and running quickly so that we can start iterating.
There is nothing wrong with focusing on speed, and we've all been in a place
where we really had no choice. I would, however, like to urge anyone
starting out on a significant project to reconsider exactly what constitutes the 
"viable" in "minimum viable product". The bar is a lot higher than you'd like to think.

<!--more-->

There are some things that all developers eventually learn. We learn that rubber
ducking doesn't work with an actual rubber duck, and that some things are
_infinitely harder_ if you wait to do them. There are a lot of crucial
components to an application that are simply left out of the first iteration.
Later, we find ourselves bolting them on as the business starts encountering a new
slew of problems that are inevitably caused by the decision *not to include them in v1*.
The following list is not exhaustive, but is comprised of examples of the sort
of things that we have all found ourselves saying "wow, this would have been so
much easier if we just had this from the beginning".

* A testing harness
* A metrics/monitoring tool
* An error reporting tool
* Tracing entry-points
* Feature flagging
* Dynamic/runtime configuration
* Secret/encrypted data service
* Horizontal scalability

These things are *not* optional for a serious product. They only *feel* optional
because we don't need them to get our hands on an MVP. Everything I listed up
there comes with associated costs. Some of them cost money, and some of them
cost time, and I've just said the same thing twice. The best way to find
yourself over-spending on technology is to scrimp on the foundation of a quality
application, and see if the money you make from your product can out-pace the
interest on your technical debt. If you've had experience with real, long-term
technical debt, you'll know that isn't a race you want to find yourself in.

> The best way to find yourself over-spending on technology is to scrimp on the
> foundation of a quality application, and see if the money you make from your
> product can out-pace the interest on your technical debt.

The important thing to note about that list, is that your v1 really doesn't need
to include a full fledged version of any of those things. What you need is to
ensure that all of them are codified as *first class concepts*. It can be as
simple as having a module or a class called `FeatureFlags` that has functions
that return booleans denoting whether or not features are flagged, given some
context. At first, there might be no required context, so maybe these are zero
argument functions. However, when product comes to us to request that a specific
user type shouldn't be given access to some feature, we don't just wrap a few
portions of our code in an if statement and call it a day. Instead the
conversation is about how we need to modify our *existing feature flagging
pattern* to fit our new needs. These discussions will always be more fruitful.
Over time, after a few new requirements have come in, we will have established a
decent pattern for enabling/disabling features based on user or environment
context. Thats all it takes. Start with a pattern, even a simple one, and you've
won half the battle.

> Thats all it takes. Start with a pattern, even a simple one, and you've won
> half the battle.

One of the things that makes these sorts of things so difficult for developers to
prioritize/focus on in the initial stages of development is that *we aren't the
ones who pay the up front cost* of leaving these things out. For instance, when
our error monitoring is inadequate, the support team pays the price. When our
tests are inadequate, QA pays the price. It isn't that developers don't care
about the other teams we work with, its that we can't adequately measure the
impact of these decisions. Talk to anyone about the support team, and they will
have stories upon stories of customer interactions and how they relate to
specific bugs. The worst days in their career will have been caused by a bug.
Ask the developer who made the mistake that *caused* the bug and he probably
won't even remember it.

If you want to have a healthy company, team, and product, focus on building
*infrastructure first*. Ignore your foundation, and you'll see it all come
crumbling down around you.
