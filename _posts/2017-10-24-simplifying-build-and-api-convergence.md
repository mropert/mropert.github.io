---
layout: post
title: Following-up on 'simplifying build in C++ (part 1)' and  discussing API convergence
tags: [build, cpp]
description: >
  My previous post about simplifying build in C++ gathered some reaction but I feel that a significant part of them seemed
  to miss the point.
  In this follow-up, I try to explain why by laying out the recipes to make several APIs converge as one.
author: mropert
---

In [Simplifying build in C++ (part 1)](/2017/10/19/simplifying_build-part1/), I tried to make the case for a unified build system
interface that could be used by package managers to ease-up their developments and tackle some issues inherent to the current
state of affairs.

It gathered a few reactions, some of which went in a surprising direction. I saw some threads derail into discussions about `build2`
(and some other similar systems) and how they could solve the problem in a much easier or better way. And that, I'm afraid,
is wrong assumption.

### The problem with the "best" solution

As some of you have pointed out, a brand new and clean build system that also integrates a package manager from the start
might seem like the best approach. Indeed, it is probably cleaner and simpler to implement than the option I'm proposing.
There's no quirky integration with legacy build systems to think about, no building upon scripts that should in all
good faith be considered technical debt, it's a clean start.

The problem with that is that C++ is not a new language. It's 3 decades old, it's got hundred of thousands (or more) projects
running in various versions of the standard, each one with its assorted build files. This makes migration a huge deal that
*must* be addressed. And that's where new & fully featured systems fail: the only migration path they offer is to rewrite
all those build files.

Worse, during the transition phase, every project that switches to the fully integrated build system will "disappear" from the rest
of the ecosystem because it's even less designed to interact with the rest of the world. It may be possible to workaround
that for single (no dependency) projects, but as soon as they rely on an integrated package manager, it will be a nightmare
to integrate them in another existing package/dependency managers. This effect will become less of a problem as more and more
projects switch to that new system (because there would be no point anymore, everything is already there)
but the fact that the first movers are the most disadvantaged would probably mean that nobody will.

After all, who would take the risk of disappearing from the (clearly imperfect) existing ecosystem to become part of a better
one if it's library is the only one there? The history of computer science has shown quite a few times that developers are much more
conservative than they admit and will think twice (at least) before abandoning backward compatiblility. Especially
if it doesn't yield a substantial advantage in the short term.

### Alternative options

Some would argue that some incentive could be provided to make a sudden move to another ecosystem feasible. I've considered two of them
but I don't feel they are possible:

1. Automatic migration  
  If there was a way to automate migration from one build system to another, then people might consider it. Advocates could
  offer pull requests easily to maintainers to speed up the process and encourage adoption of their new system.
  Unfortunately, my experience with CMake, autotools and other build systems that rely heavily on scripting in Turing-complete
  languages tells me that it's not doable. Both `build2` and Bazel offer little to no scripting feature, which is, by itself,
  a sound design choice, but makes automatic migration impossible for all but the most trivial cases.

2. Standardization  
  Should one of the fully integrated solution become part of an industry standard that's followed enough by the community, it
  could provide enough motivation for people to switch. It will take time (I still maintain C++03 code on a daily basis) but it
  could happen. The problem with this idea is that the C++ committee's current position is going the opposite way. They
  stated on several occasion that build systems should be kept out of the standard, so it's a very unlikely perspective.

With all that considered, I'm again saying that going for a common build system interface like the one I proposed is the way to go.
Since this post is still (kind of) short and I don't want to end my post talking only about build systems (again!),
it's time for the bonus post!

### Bonus post: Making APIs converge

The problem we face here is in fact not so uncommon in computer science. We have several similar services with APIs
that have been developped over time in an industry.
They all do more or less the same job but with a different interface and possibly a different paradigm.
None of them has managed to take over, which leaves our community fragmented.

We want to correct that. We want clients to be able to access all those services with only one API. We want to
create a standard.

The first thing we want to avoid in our initiative is to become yet another XKCD reference ([#927 in that case](https://xkcd.com/927/)).
If we start by introducing a new competing API, we'll just end up where we started with even more fragmentation (unless some
regulatory entity can force everyone to migrate, but most of the time it's not an option).

So we go for the next best thing: Abstraction. As David Wheeler famously said:
"All problems in computer science can be solved by another level of indirection"
(the [FTSE](https://en.wikipedia.org/wiki/Fundamental_theorem_of_software_engineering)).

What we're going to do is we're going to provide an abstraction over the existing APIs. That way we can quickly solve
the initial problem (having a common API to mix & match the services) while keeping our options open for the future.

Let's say a cool way to write services emerges, making creation easier and decreasing the cost
of maintenance. We can first add some glue to make it available through our abstraction
so that clients can use it and offer a quick way to write new services that can still work like the existing ones.
Then when the maintainers of the existing services find the time to refactor, they can migrate to the new cool API
too if they want (and when they want).

In short, we can solve both the lack of standard issue *and* provide a way to migrate without a time constraint by:
1. Creating an abstraction over the existing APIs and directing clients to it.
2. Offering the (new) best API we can think about to ease up development, and wrapping it behind the abstraction.
3. Starting new development using the new API right away.
4. Migrating the old APIs to the new one at our pace while maintaining backward compatibility.

In the practical case of my previous post, the APIs are the various build systems, the abstraction is the common build system
interface that package managers could use and the new & cool API is a simple and easy C++ build system that may (or may not)
exist yet and that I'm gonna talk about in part 2 of the "Simplifying build in C++".

So yeah, ok, that article was still a bit of a teaser for the next one, sorry about that :)
