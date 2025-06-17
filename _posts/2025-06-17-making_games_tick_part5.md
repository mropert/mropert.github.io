---
layout: post
title: "What makes a game tick? Part 5"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today we look at some more real life script performance issues and discuss the two hard problems in computer science.
author: mropert
---

[Last time](/2025/05/27/making_games_tick_part4/) we studied one real life case of script performance related to gameplay.
While it required a bit of game knowledge, and perhaps more software engineering experience than what we expect
designers to have, it was easy enough for us to fix and could be done entirely without access to code. But
sometimes studying those performance bottlenecks reveal problems that run deeper and require some re-engineering
to be properly solved. Today we look at an art bug from early Victoria 3.

You are not required to listen to the soundtrack for this episode, doing what Civilization IV did could also work if you can
put some Brahms or Dvorak on.

## To build a factory

Victoria 3 is a game about the Industrial Era. It wouldn't be complete without the audible "choo-choos" of the steam
engines and, of course, the very healthy smog they produce. This is why the game tracks how much pollution is generated
in a given area. It has a bunch of latent effects on the people living there, but more importantly for today's example
it has a _visual_ impact shown to the player.

![](https://shared.akamai.steamstatic.com/store_item_assets/steam/apps/2282100/ss_71a8276da2996d7ed5d68f31a560f231c5e5242f.1920x1080.jpg)
*Victoria 3 as seen here sans-smog on the Steam store page*

In the early days of Victoria 3 (I cannot remember if it was before or after the initial release), I ran across an issue where my FPS
would just tank, even when the game was paused. This was my first clue as to where the problem was. If the framerate was
choppy when paused, this was pretty much a guarantee it was due to graphics or UI, since the simulation threads don't run
when the game is paused. After some digging I found this in the city graphics script:

```lua
-- Recreated from memory as I do not have access to Victoria 3 at present
if pollution >= 25 and pollution < 75
    effect = city_smog_light
else if pollution >= 75
    effect = city_smog_heavy
```

At first glance this looks pretty innocuous. If the pollution is between 25 and 74 apply the light smog effect, if greater
use heavy smog, if not don't use any effect. It could have been that the effect used a very expensive shader but the
thing was the issue was present at game start, when the player barely had time to build any factory and generate enough
pollution to show. In this case this was nothing the artist had done or could have predicted. No, the issue was the `pollution` binding itself.

In code this was translated to call to `CState::GetPollution()` with `this` bound to the state (administrative division) the city
being rendered belonged to. At middle zoom level, you could be looking at a dozen cities (and states) that would need to run this code.

```cpp
int CState::GetPollution() const
{
    int pollution = 0;
    for ( const auto& building : GetBuildings() )
    {
        pollution += building.CalcPollution();
    }
    return pollution;
}
```

So the value was recalculated each call, but maybe that's a trivial thing?

```cpp
int CBuilding::CalcPollution() const
{
    int pollution = 0;
    for ( const auto& method : GetProductionMethods() )
    {
        pollution +=  method.GetBasePollution();
    }
    return pollution * CalcEmploymentRatio();
}
```

Uh oh. I'll spare you the deep dive, but given that this is a Grand Strategy Game about the industrial revolution, you can bet
calculating the employment ratio isn't a trivial computation. This is the kind of game which wants the player to ask "would it be
worse to have a shortage of machinists or bureaucrats in Nottinghamshire?". And remember we are asking this question for every building
in every state visible on screen. Up to three times if the value is over 74.

I want to highlight that unlike the previous example we brought up, this one is a case of code/script boundary where the scripter
has virtually no way of solving the problem. They are given a script binding that simply cannot perform adequately for the task
at hand (querying the pollution in a city to determine how to render it). The programming team has to step up and figure out a
way to solve this.

## The two hard problems

An over-eager coder may suggest we multithread this. After all the calculations are independent. We could use one thread
per visible city/state. Or use some sort of parallel loop in `CState::GetPollution()`. But when it comes to performance
(or software engineering in general), remember the Jeff Goldblum rule: before asking whether or not you can, ask whether or
not you should. Do we _need_ to recalculate the exact pollution score of every state each frame, possibly three times?

The obvious answer is no. Especially for graphics, but more on that later. Victoria 3 splits each in-game day in 4 ticks.
I can't remember if factory employment shifts by the day or quarter day, but I'm sure designers agree
we shouldn't bother recalculating pollution more often than daily. We could probably even get away with weekly or monthly.

But first let's look at the code again. There are two hard problems in computer science and this illustrates both: 
[cache invalidation and naming things](https://www.karlton.org/2017/12/naming-things-hard/). There was a soft rule
at PDS that methods should only be called `Get` if they performed trivial calculations or no calculation at all. Else they should
be called `Calc` to explicit the potential cost of calling it.

This obviously broke the rule, but I argued at the time that the rule wasn't enough. Because while it solves the naming
problem, it leaves the cache invalidation part entirely unanswered. It does establish that the call is expensive
and encourages us to cache it, but it doesn't give us any clue about when to invalidate the cached value. In an internal
talk I made the case that `Calc` functions should be private, and encapsulate the caching mechanism next to the logic
that performs the calculation. The rationale being that changing one _should_ imply changing the other because their
lifecycles are bound: if ones touches up how pollution is calculated, they should ask themselves if the caching mechanism
needs to be updated.

There are many ways to handle cache invalidation but broadly speaking the solutions fall in two categories: data driven
and expiry. The first one implies keeping track of all the data involved in calculating the value(s) being cached and flagging
the value as dirty when it happens, the second is to use some time or volume based metrics (every 10 seconds, every 1000 calls...)
to mark the value as stale. The first one is usually harder to implement due to the need of tracking every variable in the equation
but is always guaranteed to be up to date (providing no input data was missed), while the second is simple to implement but is likely
to yield stale results in some cases. Also the first one can result in an infinite loop if two cached values depend on each other,
I can neither confirm or deny running into some of these in Paradox games.

In this particular case (and arguably in most cases in GSGs) we elected to go with the second option, caching the pollution value
of each state and recomputing it every day (or was it week? I cannot recall). I believe we also decided to add similar
caching for some other building related values and update them daily. Those were turned into daily/weekly update tasks
that use as many threads as possible, since "calculate X for each state in the game" is the kind of computation that
parallelize really well.

## Final thoughts

In this particular case, the value was used by both the simulation and the graphics. If it had been only for the graphics we could
likely have refined the cache invalidation strategy a bit more. For example we shouldn't bother to recompute stale values until they
are needed for states visible on the screen. This would mean decoupling cache invalidation (marking data as stale) and cache update
(updating stale data with a fresh value) into two steps with different rules (first based on time and second based on visibility).
This is generally a good design pattern to maximize the efficiency of a caching mechanic.

Readers might also have encountered caching systems that amounted to the following:

```cpp
auto CachedValue::Get()
{
    if ( IsStale() )
    {
        Recompute();
        SetStale( false );
    }
    return m_value;
}
```

I would generally advise against it because while this encapsulation guarantees that values will only get recomputed when stale _and_ needed,
it makes the whole operation mutable, meaning it needs to be synchronized across threads. This not only apply to recomputing the value but also
requires locking any data accessed by the `Recompute()` function. I find it generally simpler and easier to have one explicit task that uses
all available threads to recompute instances of a cached value, by construction it should guarantee that the game objects accessed by the `Recompute`
function will not change during the process (since all threads are busy with a task that only write to one thing).

Speaking of threads, I think this is a good segue to the next big topic we should bring up: multithreading and concurrency in game ticks.
Expect several articles worth of content, see you next time!
