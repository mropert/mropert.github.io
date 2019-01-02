---
layout: post
title: Stranger in a Strange Land
tags: [build, cpp]
description: >
  A discussion flared up quite recently on social media and various blogs in which the game development
  community expressed some dissatisfaction with the general direction of the C++ language and committee
  behind it. As a game developer myself, those complaints do not really resonate with me. I try
  to explain why.
author: mropert
---

A couple days ago, Aras Pranckevičius (from Unity Technologies) wrote up a
[good summary](http://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/) of the various complaints the game
development community recently expressed about the directions taken by the C++ language.

At least part of the discussion started after Eric Niebler published
["Standard Ranges"](http://ericniebler.com/2018/12/05/standard-ranges/) earlier this month, an article
celebrating the adoption of ranges in C++20.

As summarized by Aras, the list of planned changes for C++20 felt a bit out of touch for the game
development folks who had other (unadressed) concerns in mind such as compile times, performance of
debug builds and a more general feeling that the language was going into a weird, alien paradigm.

A less productive discussion followed on social media in which some accused the C++ committee of being
an ivory tower that doesn't care about their problems while others retorted that participation was
voluntary and people just had to show up at meetings to make their case heard.

## Background matters

At that point I feel like an UFO, as I am part of the videogame industry but don't believe that
C++ is headed in the wrong direction or that the committee as lost touch with the community.
But I think it is probably because of where I come from.

I didn't start my career in videogames. My engineering thesis was on operating systems design,
then I worked for the web (in C) and then in finance (in C then C++) for a total of more than 10 years
before joining a videogame company.
Before opening the codebase of a videogame, I went to a couple big C++ conferences, spoke at some of them,
organized meetups for my local user group and of course in the process met a lot of people who are committee
members, voters, writers or simply influencers.

Armed with that perspective, I have been awaiting the arrival of ranges since Eric's keynote at CppCon 2015
and know very well that they are the continuation of research done in Boost.Ranges for more than a decade.
In short, I don't consider that direction new or revolutionary.

In the same angle, I've been to SG14 at least once, I follow the mailing list a bit and I know they
care very much about videogames and usages of C++ for low latency in general. So I do believe
my industry has a voice in the committee.

## Different strokes

Does this mean the complaints we read so far are null and void? Certainly not. But I feel some
might be misdirected.

Regarding performance in debug, I admit the usual answer I usually give is "benchmarks in debug
make no sense". C++ is a complex language of powerful abstractions through which the compiler can
see and optimize a large deal. At the cost of debuggability.

One the common justification is that with better abstraction come less bugs. It's very hard to
leak or double free a resource that uses RAII. Algorithms and ranges make off-by-one 
almost impossible and handle empty cases naturally. One the big objectives of Modern C++
is to be correct or refuse to compile, thus eliminating the need to debug. And when
everything else fails, there's still modular design and unit testing.

Still you might feel like some of your use cases are not properly handled by this. In which case
there's two things you can do. One is to ask the C++ community. Go to your local meetup or conference.
Maybe there's a paradigm that would simply eliminate the need for performance in debug? If not,
I'd suggest contacting your compiler vendor as the C++ standard leaves that matter of the implementation
and won't be able to do much unless it's a systemic issue that vendors can't solve with the present wording.

## Did someone say build?

For those who knew me before this article, you probably know that I do talk at lengths about build.
Granted, build times is usually not my number one topic, but it is partly because the problem already
has solutions.

Google famously described how their build system can handle the entirety of their codeline with extremely
fast times, and I suspect most of us mere mortals have to work with much smaller source trees.
The secret? A healthy build files hygiene, a dash of modular design and build servers. A lot of build servers.

"But I have nowhere near Google's budget for my servers" I already hear people say. It's true, but also usually
not the true issue. The amount of servers you need is proportional to your company (and codebase) size, and
experience has found that no matter how expensive it might look, it'll still end up cheaper than having developers
doing nothing while projects rebuild (or worse, hack the build system to avoid some recompilations leading to
non-reproducible builds).

No, the real problem is the cost of setting up the build system. Not the servers, the build itself. Some
products help, but having a true distributed build with reliable binary caching will most likely require
a build engineer or ten. Worse, most build systems don't enforce a proper build hygiene, leading to
horrible dependency graphs where every change requires to recompile half the project (at best).
I've made a talk about it[RINKU].

Can the committee do something about it? Not really, as build systems fall outside the C++ Standard.
But it doesn't mean their work can't improve build times through other means.
Modules TS has been a big topic for the past 5 years and when it finally reaches the standard it should
offer precompiled headers on steroids (and more). Microsoft is known to have used an internal implementation for
quite some time to speed up their builds.

## Doing one's part

While I wrote this article and thought about Dan Sacks' talk, I got the feeling that at least some of
the recent controversity come from a lack of communication between two parties.

I can't help but feel that my industry and the rest of the C++ developers should meet more.
I don't see a lot of videogame devs at meetups or conferences. They have their own conference,
the GDC, of where I can't even say what they talk about since the recordings are not public.
I haven't met a lot of coworkers who knew about Godbolt, Quickbench or CppCast.

As the communities grow apart, the dialogue becomes more difficult because the habits and techniques
start to diverge. Reading some of the discussions on Twitter reminded me of the experiences of Dan Saks
when he [tried to talk to C developers](https://www.youtube.com/watch?v=D7Sd8A6_fYU)
(I just can't recommend this talk enough!).



