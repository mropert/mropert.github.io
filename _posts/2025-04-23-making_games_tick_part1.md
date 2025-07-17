---
layout: post
title: "What makes a game tick? Part 1 - Introduction"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. And how to make them go fast.
author: mropert
---

A lot of articles I read about game development focus on performance (and also retvrning to C, but that's
another topic entirely), and yet they seldom talk about game simulation. They often focus more on graphics
and asset loading/streaming. This might be because they are often bottlenecks on AAA games, but that's not the only type
of games out there. And even for a AAA style game, I 'd argue this is an under discussed topic. At best it will
come up when talking about "large" number of AI actors. It's probably no coincidence that Unreal's
[Mass Entity](https://dev.epicgames.com/documentation/en-us/unreal-engine/mass-entity-in-unreal-engine) beta framework
is found under their "game AI" section in the documentation.

In this series of articles I would to talk game simulation performance and architecture. First we will go
through some very basics to make sure everyone is on the same page, and then we will go (much) deeper.
So let's dive in and take it from the start!

## A basic game loop

At the core, most games look like this:

```cpp
while ( !quit )
{
    handleInput();
    if ( !paused )
    {
        updateSimulation();
    }
    render();
}
```

Every iteration, the game reacts to the player input, updates the simulation to simulate the passage of time in-game, then renders a frame. And then loops back again. In practice it's a bit more complicated than that in most game engines, there are threads
involved to try and render, push work ot the GPU and update the simulation at the same time to reduce latency, but the idea
remains the same and is good enough for now. For most of this series we will be focusing on the `updateSimulation()` part,
although later articles will mention how it can interact with the other bits in multi-threaded contexts.

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
