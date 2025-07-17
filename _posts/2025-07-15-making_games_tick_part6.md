---
layout: post
title: "What makes a game tick? Part 6 - Threading Models"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. In this episode we go back to code architecture and check out a few approaches
  to using multiple threads.
author: mropert
---

After a [three](/2025/05/07/making_games_tick_part3/) [part](/2025/05/27/making_games_tick_part4/)
[digression](/2025/06/17/making_games_tick_part5/) about scripting, it is time to revisit our first two articles.
[Back then](/2025/04/30/making_games_tick_part2/) we mentioned how some of the common engines on the market handle ticking
in a somewhat crude `virtual void Tick() {}` fashion. Today I wanna show some alternatives I have used when
making Grand Strategy Games.

Readers who have seen my talk [Multi Threading Model in Paradox Games: Past, Present and Future](https://www.youtube.com/watch?v=M6rTceqNiNg)
might recognize a few things I brought up in the past in video form.

## The board game way

Grand Strategy Games started out as boardgames. Insane boardgames for crazy people. If you think I'm being mean,
look at how Europa Universalis looked like in this original form:

![Paper Universalis](/assets/img/posts/paper_universalis.png)

No wonder someone saw this, thought "there has to be a better way to do this" and then remembered computers exist.
(I do not have a direct quote to prove this is how the first GSG happened, but it is my headcanon).

If you've ever played a board game, you're probably familiar with the general flow where each player takes a turn,
and each player turn is usually split into several phases (incidentally, some of the more modern ones have implemented
simultaneous turns, human multithreading in a sense). Owing to that heritage, that's how games like Europa Universalis
were often done back then. The English player takes a turn, moves their units, collects taxes, progresses their constructions,
then passes the baton to the French player who does the same, then Austria, then the Ottomans, all the way to the last "player"
(which would also include any nation controlled by AI, and there's _a lot_ of them).

This is not very multithread friendly as an approach, but when making games in the 2000s this was of least concern. 
From my research at the time, the first of parallel algorithms for game simulation happened some time during a later
Victoria 2 patch and/or during the development of Crusader Kings 2, in the very early 2010s. As a reminder, the
first consumer dual core CPU, the Pentium D, only came out in 2005. And games usually don't use  the latest and most expensive
product as their baseline. It's bad for sales when only a fraction of your playerbase can afford to run the game.

As the 2010s rolled around, Grand Stategy Games were ticking by updating each collection of game objects in turn.
Roughly speaking, a tick in a game like Europa Universalis IV or Hearts of Iron IV would look like this:

```cpp
void World::Tick()
{
    UpdateCountries();
    UpdateProvinces();
    UpdateArmies();
    UpdateNavies();
    UpdateAI();
}
```

"Where's the thread usage?" our players loved to ask. Within each collection update. To varying degrees.
Usually we'd split each one of these functions between a parallel and a serial pass. It's up to the implementer
to figure out how much they can lift into the parallel pass rather than in the serial pass. The trick
was to find out what could be parallelized (or more precisely, rewritten to work in parallel) and what could not.
Sometimes it was as simple as replacing a `for()` loop by a `parallel_for()` loop and see if anything exploded.
Alternatively we would need to go down the rabbit hole of each function call to see whether or not it was safe
to update the finances of two countries at the same time.

## Does it have to be this painful?

The reason this was so hard to achieve goes back to the running theme of this entire series: implicit data writes.
Two functions can only run in parallel if they don't have a data race, meaning they cannot share a variable with
read/write access. Read-only sharing is fine, which is in my opinion the reason why C++ (rightfully) insists
on using `const` everywhere. This was already discussed back in 2012 in Herb Sutter's talk
[You don't know const and mutable](https://www.youtube.com/watch?v=1aT2UVfwPUY) (thanks anonymous user for
putting a reupload on Youtube since Microsoft took down Channel9 and made all the C++ blogs point to dead links).

![How do I know that this function is thread safe? That's the neat part, you don't!](/assets/img/posts/neat_threads.png)

As we hinted in a previous article, being able to tell whether or not two functions (or two invocation of the same function with
a different argument) run in parallel won't create a data race is really hard. If you add the facts that most games
allow implicit read/write access to the whole gamestate or can invoke script which will, it becomes near impossible
as the codebase grows in size. As mentioned before, it might become simpler to just try to parallelize a loop and
see if anything breaks.

The large-scale architectural solution to this issue would be to do away with implicit data accesses and make them all explicit,
that way we could guarantee at compile time (or at worst with an assert) that a parallel pass does not invoke a data race.
And while I believe this is the way forward for the future, this might require a partial (or complete) rewrite of the game logic.
I will explain it more in details in the next post (or the one after that), but for now let's focus on more iterative solutions.

## The modern medieval approach

Already in Europa Universalis IV, we had an interestingly named function called `UpdatePrivateOnly()`. The idea
was, this function (and all its callees) was only allowed to write to private variables that could not be
accessed through any public interface. In fact, only the companion `UpdateSelfDontReadOthers()` was supposed to access those variables.
A first pass would run the first function in parallel on all objects in a collection, then we would do the same with the second. The first one
could read anything, but only write to a designated set of variables, while the second could write to any field
in the object but wasn't allowed to read from siblings.

```cpp
void World::UpdateArmies()
{
    // Compute stuff and write it to the private area, don't modify anything visible
    std::for_each( std::execution::par_unseq, begin( armies ), end( armies ), UpdateArmyPrivateOnly );
    // Copy data from private area to the public data members
    std::for_each( std::execution::par_unseq, begin( armies ), end( armies ), UpdateArmySelfDontReadOthers );
    // Serial pass
    std::for_each( begin( armies ), end( armies ), UpdateArmySerial );
}
```

I don't believe I ever came across a catchy name for this pattern, but you could see it as some form of local double buffering.
Although this was not formalized as such and not every data member in the private area had a 1:1 public member to match,
there's certainly a family connection between the two. Fans of functional programming within my readers will likely be happy
that we managed to reinvent the basic idea of pure functions from first principles. 

My former colleagues working on Crusader Kings 3 made two solid improvements to this idiom. One, it was made more
systemic so that every game object collection would allow it and it would be easier to teach to new team members:

```cpp
template <typename GameObject, typename TempData>
struct Manager
{
    std::vector<GameObject> objects;
    std::vector<TempData> temp;

    void PreUpdate()
    {
        std::ranges::for_each( std::execution::par_unseq, std::views::zip( objects, temp ), PreUpdate );
    }

    void Update()
    {
        std::ranges::for_each( std::views::zip( objects, temp ), Update );
    }
};

// Specialization for Armies
void PreUpdate( std::tuple<const Army&, ArmyTempData&> );
void Update( std::tuple<Army&, const ArmyTempData&> );
```

This way, the function signatures make it a bit more explicit that every game object has an associated data buffer
in which the results of expensive computations be can stashed for later. "When possible, do the heavy work in `PreUpdate()`
then copy/apply the results in `Update()`".

The second big change was the realization that if the rule forbids the `PreUpdate()` pass from writing
_anywhere_ outside of its designated stash, then there is no need to split the execution by object type. They could naturally
all run at the same time:

```cpp
struct ManagerBase
{
    virtual void PreUpdate() = 0;
    virtual void Update() = 0;
};

template <typename GameObject, typename TempData>
struct Manager final : public ManagerBase
{
    // ...
};

std::vector<ManagerBase*> all_managers;

void World::Tick()
{
    std::ranges::for_each( std::execution::par_unseq, all_managers, PreUpdate );
    std::ranges::for_each( all_managers, Update );
}
```

This change allowed two things: one it improved parallel pass thread utilization by making sure that if a worker thread
queue ran out of countries to process, it could grab some provinces or armies to work on. Two, since there
was functionally _no visible state_ that would mutate during the `PreUpdate()` parallel pass, it wouldn't need
to lock the gamestate to run, meaning it could run simultaneous to the rendering.

## Read-only update

I haven't brought render locking since the [very first post](/2025/04/23/making_games_tick_part1/) but this is another
big issue when considering tick performance and multithreading: the `Tick()` jobs aren't just fighting with each
other for data access to the game objects, they are also fighting with the renderer that is trying to display the state
of the world 60 times per second, and cannot do it if any other thread is writing to it. Thinking of it, this
sounds like a topic that would require its own post and we've been going long again.

Let us conclude this article with the idea that there are likely a breadth of systems in a game loop (both in world tick
and in rendering) that functionally can all run simultaneously if they do some sort of double buffering, and that change
alone might be enough to improve a game's performance by a lot. Next episode, we will continue with the topic by bringing up
a few issues that may come with this and how to solve them. This is also your chance dear reader to bring up any challenge you may have
faced if you ever tried something similar in whatever social media platform you got this article from.

Until next time!
