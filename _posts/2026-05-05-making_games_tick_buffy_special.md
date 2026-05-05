---
layout: post
title: "What makes a game tick? Special Issue - Buffy the Performance Slayer"
tags: [cpp,gamedev]
description: > 
  Let's talk about game simulations. In this special issue we talk about how to handle buffs,
  modifiers and other stat bonuses.
author: mropert
---

Whether one is making a RPG or a strategy game, there usually comes a time where designers want
to attach a bunch of stats buffs and debuffs to each and every object in the game.
The game actors start small but eventually we want our characters, units, countries and monsters
stats to be able to be affected by a mix of equipment, perks, area effects, difficulty settings
and whatnot. And if we don't take care, it might turn our game into buff recalculation simulator 2000.

For those who never had design or implement a stats system, this might not sound like a very hard problem
at first glance. After all, shouldn't it simply be a simple `struct` with a bunch of values in it?
It could, but it depends a lot on what your system can and cannot handle.

![Average country buffs in EU4](/assets/img/posts/eu4_buffs.png)

Say that we want our actors to have a bunch of stats that can be affected by various sources as we mentioned in
our intro paragraph. Let's ask a few questions about the extents of our system.

## To cache or not to cache?

First let's get the most basic question out of the way: "do we need store the sum total of a given stat
with all buffs applied?". After all, we could simply recalculate it on the fly each time. This helps
us framing the problem domain as fundamentally a caching issue. It's often said that cache invalidation
is one of the two hard problems in software engineering, so can we dispense with it?

This might sound like a no-brainer but it actually depends a lot on the game, and then possibly on the stat
itself. It forces us to ask ourselves (and the designers): how often does this value change compared to how often
it is used. This is made trickier by the fact that the answer might change as the game design evolves.
Maybe the designer has great plans for a given stat, but it turns out "bonus torpedo speed for submarines
in deep waters" isn't a value that is often used in the game<sup>[1](#myfootnote1)</sup>, or maybe a stat started as a niche thing,
but became ubiquitous over the years.

On a similar note, how expensive is the value itself to compute? Is it just about querying a couple values
and summing them? Does it need to iterate over collection of objects from possibly dozens or thousands of potential
contributors? And finally, does it need to recursively query values from another game object that will likely
be using the same system?

Assuming we see a need for caching at least some class of stats, let's continue with our next question.

## Source tracking

Another very important question is "do we need to keep track of the contributors to a given stat?".
In the most basic case it's not necessary. We could simply have a value, add to it when a buff is activated
and subtract from it when they're disabled (for example when a character equips or unequips a piece of
equipment).

I say very important because it can easily be overlooked in the early stages of development. At first glance
it looks like it won't be necessary. The architecture gives us clear `OnEnable()` and `OnDisable()` callbacks
that we can use to make sure the value is up to date. The designers don't foresee the need to track where
a buff comes from. They even thought about stacking rules and concluded that it's fine to stack multiple sources
so we won't need to make sure only the highest spell effect should apply if multiple are present.

But then the first beta test feedback comes back, and the most common comment is that the UI doesn't
show the breakdowns and it's impossible to understand why a given stat sums up to a given value. Or maybe
QA complains that it's hard to test whether or not the buff system actually works and requests at least a
debug tooltip explaining how the value was calculated.

So... does it mean we should always keep track of breakdowns in our stats caching system because we'll
always need to at least display it in some form? From my experience, I'd argue mostly yes, although
I've considered cheating once or twice. Given how expensive the whole system can end up being<sup>[2](#myfootnote2)</sup>,
one could consider having the UI breakdown code path bypass the caching system and redo the calculation
on the fly, since we usually only display a handful breakdowns per frame. The displayed value can end
up slightly out-of-sync with the cached value that is actually used by the tick updates, which can confuse
players and testers alike. If you can afford it, not tracking buffs sources in the cache will save precious
CPU cycles (and memory).

The one thing I definitely don't recommend is forcing a cache recalculation on the fly (with breakdowns)
when the UI needs it. I have fixed several out-of-sync MP bugs in my career because of it.
And even for single-player only games, this introduces the ability for the UI to write to the gamestate on the fly,
which if you've followed my previous articles should know that I'm very much against.

## Pilot

With that out of the way, let's talk about how this article got started. I recently had the opportunity
to consult for Galactic Starfish on their upcoming title Strategeist. One of things I noted is that,
much like every Grand Stategy Game I know of, there was a performance issue with their modifier system.

They kindly allowed me to use their source code as a "real-world" benchmark to test out various
implementations for the purpose of this article<sup>[3](#myfootnote3)</sup>. I wouldn't take the current numbers as any indication
of how the final release might eventually look like, they are only given to illustrate the impact of
various implementation choices.

So, let's start with our baseline implementation:

```cpp
using StatID = uint32_t;
using BuffSource = uint64_t;

struct Stats
{
    struct Entry
    {
        double value;
        std::unordered_map<BuffSource, double> sources;
    };

    std::unordered_map<StatID, Entry> entries;
};
```

Pretty straightforward so far. In the case of Strategeist there is at the moment 480 unique stats in the game.
Grand Strategy games usually get into the thousands. Those should probably be split up in categories (by which
final game object classes they can apply to), but still that's probably a good representative number.

Now as I mentioned before, this implementation was fairly expensive to run. On my i7-10700, I measured the cache
recalculation of each in-game country (the actor the player controls) to around 163.7us per run. Of course
this is done for each country in the game, and if you're familiar with map games you already know that there are
hundreds of them in the game.

While there are probably things that can be improved in the calculation themselves, for the rest of this article
I want to focus on how much could be achieved by just changing the underlying storage.

## So you're like a good demon? Bringing allocations?

While I first used Unreal Insights as an instrumentation profiler to see the modifier update stand out in the tick profile,
switching to a sampling profiling (even one as basic as the one embedded in Visual Studio) was enough to confirm my suspicions:
using those containers sparked _a lot_ of allocations to the point that most of the update was spent in `new` and `delete`.
Unreal's `TMalloc` is better than the default `malloc` provided by Windows' CRT, but it was still a major issue.

As far as associative containers go, `std::unordered_map` isn't great. It's [better on MSVC than on Clang/GCC due to the way it avoids modulo](https://www.youtube.com/watch?v=M2fKMP47slQ)
but it's still not great. Especially because the issue here isn't lookup time, it's insertions/removals that trigger a bunch of `new` and `delete`.
This is due to the fact that `std::unordered_map` is basically implemented as `std::vector` of `std::list`, with each entry being it's own
heap allocated `std::pair`.

Looking at usage patterns by going through the code I noticed that we barely make use of inner `std::unordered_map` in a way that matters.
The need for O(1) lookup of a given `BuffSource` in a given entry is rare, mostly limited to UI. Most entries will only have a handful sources,
with the maximum case being maybe a dozen or two.

As it turns out, modern CPUs are actually quite good at brute-force linear search through small arrays. We could turn the inner
`std::unordered_map` into a `std::vector` and use `std::find_if` to find the right pair. If we really needed it, we could even make it
two vectors (one for keys, one for values), which could turn our `std::find` for a given key into a handful of SIMD `uint64_t` compares.

Making our `sources` into a `std::vector<std::pair<BuffSource, double>>` already yields quite impressive results: 91.8us.
It also lowers the memory footprint (`vector` is more memory efficient than `unordered_map` for the same size/capacity).

## Trying out Unreal types

Since we're using Unreal Engine 5, we might as well try out their own `vector` alternative. After all there's always people in my comments telling
me that the STL is bad and every gamedev™️ should rewrite it even on a solo project. I know I'm being cheeky but this is a recurring thing I've
been dealing with for years. In my book the bare minimum Unreal should do to win me over would be to work as a drop-in replacement of `std::vector`
(like EASTL does), but sadly it doesn't. Iterators are not copyable for some reason (which makes it unusable with most of `<algorithm>`),
and none of the methods names match the STL besides `begin()` and `end()` (if they exist at all).

Either way, let's bite the bullet and try replacing our `origins` once more with `TArray<std::pair<BuffSource, double>>`.

The result gives us 86.6us on average. That is 5us better. This is not bad, but I'm not sure it's worth the hassle especially when the main
performance improvement can be gained without it.

See, the main reason why this is faster is that Unreal's `TArray` allocates at least 4 elements when it grows from 0 capacity (unless you set a flag asking it 
to be as conservative as the STL). This avoids the common pitfall with `std::vector` that it reallocates when going from capacity 0 to 1, then
1 to 2, then 2 to 3, and (if unlucky) from 3 to 4 due to the way the geometric growth factor math works. I agree with Unreal devs that for most cases this is probably a better
strategy. In particular, I would love if the STL had a basic heuristic to avoid the 0 -> 1 -> 2 -> 3 -> 4 reallocs under a certain element size.
Worst case scenario, it can be added on our side with a simple `if (v.capacity() == 0) v.reserve(4);` check.

Either way, there's something even better than 1 allocation for small size. That's no allocation. This is usually called a small vector
(because it does Small Buffer Optimization, like `std::string`).
They aren't in the STL (sadly) but can be found in common libraries like [Abseil](https://github.com/abseil/abseil-cpp/blob/master/absl/container/inlined_vector.h)
and [Boost](https://www.boost.org/doc/libs/latest/doc/html/doxygen/boost_container_header_reference/classsmall__vector.html).
Or since we're using Unreal, we can use their `TInlineAllocator` with our [`TArray`](https://dev.epicgames.com/documentation/unreal-engine/array-containers-in-unreal-engine):

```cpp
using Sources = TArray<std::pair<BuffSource, double>, TInlineAllocator<4>> sources
```

This brings us down to 69us per update by avoiding allocations entirely for origins for most entries, unless they have a lot of buffs associated
with them, in which case the geometric allocation formula kicks in, making sure we only (re)allocate 1 or 2 times as we grow the array.
We're already twice as fast as our baseline!

## Arrays, arrays everywhere!

Now you may be thinking: "well if the inner container is faster as an array, shouldn't we also do it for the outer container?".
Congrats, I thought the same. Because that what's been done on games like Europa Universalis IV and Hearts of Iron IV.
Except there's a catch.

```cpp
static constexpr std::size_t MaxStats = 480;
std::array<Entry, MaxStats> entries;
```

... and our average stats recalculation becomes 1142.8us. One entire millisecond per actor. With several hundred actors
we're looking at a tick that may take up to a whole second. It's a disaster! What's gone wrong?

The issue is that the calculations make use of temporary `Stats` variables before adding them to the main country stats.
Each time they allocate about a hundred kilobytes. Even if they carry only a few values with one source each. While
this probably indicates an over-reliance on temporary variables that could use a re-architecture, it does illustrate a point.
Maybe not all stats need to use a large (but mostly empty) storage. Some systems and game objects with few actual entries
would probably still benefit from the sparse storage offered by `unordered_map`.

So what if we left the storage strategy up to the user? After all, they probably know better which is best in a given corner
of the codebase.

```cpp
using ArrayEntries = std::vector<Entry> entries;
using MapEntries = std::unordered_map<StatID, Entry>;
std::variant<MapEntries, ArrayEntries> entries;
```

That way, the default behaviour is to use the sparse map storage that is optimized for objects that only carry a few active stats,
while bigger objects (like countries) can switch to the array version. Note that we switched from `array<Stats, 480>` to `vector<Stats>`
to ensure the size of an empty `Stats` object remains minimal. We will allocate _once_ for objects we switch to the `vector` variant,
a perfectly acceptable cost that is paid only once upon init/construction.

The results actually show a huge jump in performance: 45.1us. We're now almost 4 times faster. Not only are lookups/inserts much faster,
but also since we now have a `vector` as base we can make sure we don't free any memory upon clear. The `origins` array under each
`Entry` will never need to allocate for most countries after one tick, because we will keep the capacity untouched. This is one of
the big advantages of dense arrays as they can easily preserve inner container allocations (it's not impossible to do with maps
but you would need a custom allocator that reuses freed entries and keeps their origins array allocated).

## Better maps?

We've been saying `unordered_map` isn't very good, so what if we tried something else? Continuing the Unreal Engine theme,
I considered using [`TMap`](https://dev.epicgames.com/documentation/unreal-engine/map-containers-in-unreal-engine) but sadly
I found the API really terrible, especially when trying to replace `unordered_map` for a quick test.
Instead, I decided to use [`ska::bytell_hash_map`](https://probablydance.com/2018/05/28/a-new-fast-hash-table-in-response-to-googles-new-fast-hash-table/),
by the author of the previously linked C++Now talk on hash tables. If you're curious about more options I found
[this article](https://martin.ankerl.com/2022/08/27/hashmap-bench-01/) offers a good overview.

Since those are a drop-in replacement for `std::unordered_map` (mostly, remember inserts invalidate iterators 😉), it's easy to try-out:

```cpp
using MapEntries = ska::bytell_hash_map<StatID, Entry>;
```

This bring us our best timing of the whole experiment: 42.4us. The small scale of the improvement is mostly due to the fact
that our country `Stats` container is using the array variant, so it only improves the temporary variables and friends used
in the recalculation code.

If we switch off the array storage for countries and use the `bytell_hash_map` in all cases, we still get an honorable 53.43us.
It also gives us a hint about the potential improvements we could get by improving our calculations' usage of `Stats` outside
of the ones stored inside the country.

Again all those numbers are given to illustrate the difference container choices can make without changing anything else
in our stats/buff system. The relative ratio is probably a bit off because the baseline calculations could be improved<sup>[4](#myfootnote4)</sup>. 

## Final numbers

During this little experiment I've tried many combinations. I'll leave CPU and memory usage (size of `Stats` per country, including
sub allocations) for reference:

- V1 (baseline) (unordered_map + unordered_map): 163.7us / 17918 bytes
- V2 (unordered_map + vector): 91.8us / 11629 bytes
- V3 (unordered_map + TArray): 86.6us / 18645 bytes
- V4 (unordered_map + TArray Inline): 69us / 18291B bytes
- V5 (vector + TArray Inline): 1142.8us / 994224 bytes
- V6 (variant<unordered_map,vector> + TArray Inline): 45.1us / 45096 bytes
- V7 (variant<bytell_hash_map,vector> + TArray Inline): 42.4us / 45096 bytes
- V8 (bytell_hash_map + TArray Inline): 53.4us / 25086 bytes
- V9 (bytell_hash_map + TArray): 71.4us / 19558 bytes
- V10 (variant<bytell_hash_map,vector> + TArray): 44.8us / 25314 bytes

By the way, I have not mentioned multithreading so far for a reason. While stats cache update is definitely something that
can trivially be thrown at a `parallel_for` for each actor, I wanted to focus on single-core performance for the purpose
of this article. Especially because, for those out there who are still using the default `malloc()` implementation from MSVC,
you will feel the pain of trying to parallelize an operation that is mostly bound by allocations. As I mentioned in
[my talk last year](https://www.youtube.com/watch?v=74WOvgGsyxs), the default allocator uses mutexes which will make all your
numbers explode. If your application doesn't already have a custom general purpose memory allocator like Unreal does, consider
switching to [mimalloc](https://github.com/microsoft/mimalloc)<sup>[5](#myfootnote5)</sup>.

And with that being said, see you next time!

---

<a name="myfootnote1"><sup>1</sup></a>: This may or may not be inspired by a game I previously worked on

<a name="myfootnote2"><sup>2</sup></a>: Both Stellaris and Victoria 3 were at some point or another bound by how fast they can recalculate stats/modifiers

<a name="myfootnote3"><sup>3</sup></a>: In general it's way too rare that game companies share any code outside of a few outliers. If you're reading this and are in
a position to change this, please encourage your company to open-source past titles for the purpose of education if anything.

<a name="myfootnote4"><sup>4</sup></a>: I've improved performance on several live GSG titles over the years by removing temporary copies of `Stats`, believe me when I say
it can make quite the difference.

<a name="myfootnote5"><sup>5</sup></a>: Yes I know, `mimalloc` is made by Microsoft. Ironic, don't you think?