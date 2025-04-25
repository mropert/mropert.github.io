---
layout: post
title: "What makes a game tick? Part 2"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today we ask ourselves basic architecture questions.
author: mropert
---

Most game engines handle simulation tick calling a overridable method on each game object. Why is that?
And is that the only approach out there? Isn't it odd that a very objected-oriented approach is the default
in an industry that [keeps complaining that OOP is bad and we should do data driven instead](https://www.youtube.com/watch?v=rX0ItVEVjHc)?

In this series of articles we talk game simulation performance and architecture. In the
[previous episode](/2025/04/23/making_games_tick_part1/) we went over the basics of a game main loop and
how it often implemented (although in a simplified manner for the sake of getting started). Today
we take a step back and think about flow and architecture.

## Why so abstract?

When I first had a look at commonly used engines (such as Unreal, Unity and Godot) and the way they handle ticks
it felt pretty weird and alien to me. I have spent most of my time in game development working with custom engines,
especially with the Clausewitz Engine at Paradox Development Studio (don't bother looking it up on the internet,
most pages I could find about it are out of date, completely wrong, or both). Let me explain by bringing back
the basic sample from our previous episode.

```cpp
// Slightly less rudimentary simulation update
static std::vector<std::unique_ptr<GameObject>> allObjects;
static std::chrono::steady_clock::time_point lastUpdate;

void updateSimulation()
{
    const auto now = std::chrono::steady_clock::now();
    const auto deltaTime = lastUpdate - now;
    for ( auto gameObject : allObjects )
    {
        gameObject->tick( deltaTime );
    }
    lastUpdate = now;
}
```

Now say you are new to the codebase and you ask yourself "what does the game simulation do?". You look at this
and the only thing one can tell is... "I have no idea". To be able to tell, one would have to look up the implementation
of each `GameObject::tick()` override and then make a guess about the order they are evaluated (by type? by spawn timestamp?)
and then another guess about which ones are active and which one may not.



## Ticking the world

Updating the simulation is often called "ticking". The word is unrelated to the bug of the same name (although it's often
the cause of them), it's related to the passage of time. Think of a clock ticking. The world "ticks" each time the clock
arm moves one unit. Which in turns notifies each game object (the objects that participate in the game simulation, such as
players, monsters, npcs, vehicles and environment objects with physics attached).

```cpp
// Rudimentary simulation update
static std::vector<std::unique_ptr<GameObject>> allObjects;

void updateSimulation()
{
    for ( auto gameObject : allObjects )
    {
        gameObject->tick();
    }
}
```

If you look at publicly available game engines like Godot, Unity and Unreal you will notice something similar (although
usually more complex in implementation).
Godot game objects have a `_process()` virtual method called each frame. On Unity it's called `Update()`. And `Tick()`
on Unreal. And while game engines can offer a smarter way to update than calling each object in serial, this is a good starting
point to reason about ticks.

## Real-time vs game time

An astute reader (or someone who's already versed in game development) may have noticed an issue in this model already.
The frequency of updates is tied to the frame rate. This has been an issue for a long time, although not as long as one
may think. Older consoles and arcades had only one hardware variant and no time-sharing operating system, meaning the
framerate/tickrate could sometimes be consistent enough to be relied upon without an extra timer.

A common anecdote, the original Space Invaders from 1978 didn't originally intend to have the difficulty ramp up as the game went.
It turned out to be a side effect of have less monsters to update and render, making the game tick faster. If they had
Jira back in the 70s, the bug report would have been closed as "Working As Designed".

One very common way to solve this today is by using "delta time":

```cpp
// Slightly less rudimentary simulation update
static std::vector<std::unique_ptr<GameObject>> allObjects;
static std::chrono::steady_clock::time_point lastUpdate;

void updateSimulation()
{
    const auto now = std::chrono::steady_clock::now();
    const auto deltaTime = lastUpdate - now;
    for ( auto gameObject : allObjects )
    {
        gameObject->tick( deltaTime );
    }
    lastUpdate = now;
}
```

This way, each object can update independently of the frame rate and also possibly absorb spikes. Physics
in particular are often expressed as "some formula divided by deltaTime". In practice the deltaTime
is often expressed as a fraction of 1s `float` instead, which allows for multiplication instead of division
in calculations, a common form of [strength reduction](https://en.wikipedia.org/wiki/Strength_reduction).

## The bigger picture

With those basics out of the way I can leave you with some thoughts while you wait for the next articles in the
series. Over its course we will try to answer two questions especially:

1. How can one reason about a game tick update? Especially build a mental model of what happens and in which order.
2. How can we optimize the computation of a game tick? In particular, can we throw multiple threads at the problem
  and if so, how?
3. Can we find a way to render the current state of the simulation while simultaneously computing the next state?

Until next time!
