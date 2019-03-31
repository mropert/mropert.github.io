---
layout: post
title: How do you keep up with tech in your game?
tags: [cpp,gamedev]
description: >
  While the majority of videogames have a relatively short lifespan, some are still actively
  developed for years or even decades after their initial release. Let's see how they can keep
  up with new techs.
author: mropert
---

World of Warcraft was initially released in 2004. Wikipedia tells me the latest expansion was
published last summer, 14 years after release. The original game could run on Windows 98. Today
it requires both hardware and software that didn't exist at release.

While WoW is certainly one of the most extreme examples, it's not the only videogame to have
an active lifespan of half a decade or more. While MMORPGs and other multiplayer games fill a good
part of that category, they are not alone in here.

At Paradox, we keep releasing new content for our strategy games more than 5 years after release.
The title I'm working on, Europa Universalis IV, will get a new expansion pack later this year,
while it celebrates its 6th birthday.

People are sometimes surprised to hear me say that the build uses C++14. After all, ISO C++14 was
not standardized at the time of release, and even C++11 support in compilers was a bit wonky when
the initial development started.

## Don't fix it ain't broken?

The first question my readers might ask is why bother upgrading at all? After all, once we made
a release with a given set of tech and know it works, using anything else will come with a cost
and a risk, right?

In fact, it's a bit more complicated. The longer development will continue on a given title,
the more it will cost to not upgrade.
New patterns demonstrated in the field will not be usable, programmers will ask to be moved to
a "newer" project, interviews will feature uneasy talks about the age of the compiler and one day
a vendor will deprecate some of the technologies used.

To give some concrete examples:
* A title released 5 years ago is unlikely to use C++11, as compilers during its development didn't
  have a great support for it, especially on Windows/MVSC platforms. Using C++03 today will be a 
  hard sell to programmers, especially new hires.
* 5 years ago, around 20% of Steam users were still running 32 bits OS, making x86_32 a likely
  target for a release. The next release of OSX will not support 32 bits binaries.
* Metal was only released in 2014, followed by Vulkan in 2016, making OpenGL the obvious option for
  OSX and Linux at the time. In 2018, Apple announced the depreciation of OpenGL in favor of Metal.

## The cost of upgrading

Over the course of a year, our team has done a bunch of tech upgrades to our title. Here's a short
overview of the efforts it required.

### Compiler upgrades

Moving to MSVC 2017 was a pet peeve of mine. While it hasn't been rolled into the main branch yet,
I've been toying with a test branch for some time. Most of the effort was spent on rebuilding
pre-built 3rd parties that were not binary compatible. Since the project doesn't use a package
manager (a shame, I know :)), it took a day or two to do. I've shamelessly used a local install of
Conan to generate a new build of OpenSSL, then spent the rest of the time fighting the build of
libvpx.

### Operating system upgrade

Our Linux target has been Ubuntu 14.04 LTS for a long time. The next patch should move that 16.04
LTS. While it was mostly painless, we had to deal with the change of the default C++11 ABI in
libstdc++. Again it was an issue with prebuilt binaries but fortunately we did have some for
both targets, so it boiled down to making the CMake finder smart enough to pick the right one.

OSX proved a bit more of an issue because (in my experience) backward compatibility and building
for an older release is badly documented. For example, did you know that setting an OSX deployment
target silently switches your runtime from libstdc++ to libc++ past a given release? The solution
for me was to let Xcode set its defaults and ensure we used a prebuilt that used the same target.
Sadly the OSX deployment target is usually not considered in platform triplets (it should), whether
it's a binary distribution of a 3rd party or a package manager like Conan or vcpkg.

### C++ Standard Upgrade

One last tech upgrade I've toyed with but not pushed yet is a full C++17 build. Of course it first
required some of the previously mentioned upgrades. On Windows it means switching to MSVC 2017.
On Linux it means using a newer Clang or GCC and linking the C++ runtime statically. OSX is
trickier since only the latest 10.14 (Mojave) deployment target will toggle the compiler to C++17.
I could try to force it anyway but Apple goes at great length to make it hard, so I'm not 100% sold
on the idea. The alternative would be to bump the minimum system requirement to the latest version
of OSX for all users. While they tend to upgrade much faster than on Windows or Linux, it's still
not an easy decision.

Some third parties proved a bit too old to handle C++17 properly. 99% of those were due to the use
of the deprecated `std::auto_ptr`. When I could, I simply upgraded the 3rd party in question. One
of them wasn't maintained anymore, but fortunately was not a big one so I could simply replace
all uses with `std::unique_ptr` manually. In hindsight, had I started the work with Linux,
I could simply have run `clang-tidy` with `modernize-replace-auto-ptr` and let it do the work for
me. In theory it could be done on Windows too, but I had bad experiences in the past.

Finally, there was the question of the game code itself. My readers will be pleased to hear that
EU4 doesn't use digraphs or trigraphs, so again `std::auto_ptr` was the only serious obstacle
and was swiftly dealt with using the same techniques described in the previous paragraph.

### Migration to 64 bits

Moving from 32 to 64 bits involved a lot of stuff that I've already discussed, from upgrading
and rebuilding 3rd parties to changing operating systems. Code wise, the largest issue we had
by far was the use of `long`. In theory, `long` would be a better choice than `int` since C and C++
only guarantee 32 bit integers with `long` (`int` only needs to be 16 bit long to be compliant).
In practice however, `int` is 32 bit on all x86 platforms, whereas `long` will be 32 or 64 bits
depending on platform (LP vs LLP).

A good way to make your project future proof on that matter is to follow this guideline:
* When doing serialization (files or network), only use `<cstdint>` types
* Everywhere else, use `int` unless you have a very good reason not to
* Never use `long` on x86 (`long long` is OK though)

### Third party upgrades

Some of the third party software we use was upgraded not because it blocked another upgrade,
but simply because a newer version brought some benefits. An update of Intel TBB fixed some
aggressive CPU usage on idle threads, an upgrade of PhysFS gave us better loading times, and
an update of the Steam SDK fixed some bugs in the in-game browser on Linux.

Once more most the challenge here was fighting CMake and the absence of a package manager, but
some also required to adapt our code to newer API. Fortunately the one that changed the most (PhysFS)
was already hidden behind an abstraction layer so the impact was quite limited. That example
illustrates some good judgement calls from the engineers who preceded me on when to use an API
directly and when to wrap it.

My personal rule is usually to look at the quality level and stability of a 3rd party API. I am fine
with using Boost directly, but I would clearly wrap anything that looks like a C API, or that has
changed several times in the past already.

## Client targets

One that I feel gets too little attention is how to decide of a given target. How can we know that
moving to 64 bits will be OK with our users? To change minimum required CPU or operating system?

To me the answer is to have some form of telemetry. Collecting some anonymous data about your users
(with the proper GDPR consideration) will allow you to know when it's safe to push an upgrade on
them. Steam may publish monthly surveys, but they are about all users, not your users. Players who
mostly play the latest generation AAA title will not follow the same patterns as those playing indie
pixel art games.

For example, on EU4 we made sure than more than 99% of our users were running a 64 bits OS before
deciding to drop 32 bit support on the next patch. Furthermore, with some data from the CPUs
of users still running a 32 bit OS, we could confirm that most of them would be able to simply
upgrade their OS to a 64 bits variant without changing hardware to continue playing our game.
In the future, this could also help us decide when to enable AVX and other CPU specific
optimizations.

## Wrapping up

The number one lesson I derive from this experience should not surprise my usual readers: having
simpler build files and using a package manager would have saved me quite some time on those
upgrades. Someone who hasn't spent some time debugging CMakeLists (don't we all?) would probably
have been stuck even longer.

So again, I'll strongly suggest you keep your build files simple, use a toolchain description and a
package manager.

I will also suggest something possibly unpopular: have some form of telemetry in your product. It
doesn't have to "spy" on users. Keep it simple and anonymous, be mindful of GDPR. But remember that
if you don't know about your users' hardware and software, you are likely to break something for
them in an update, or postpone updates fearing that your might.
