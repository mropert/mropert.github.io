---
layout: post
title: "What makes a game tick? Part 2"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today we ask ourselves basic architecture questions.
author: mropert
---

Most game engines handle simulation tick by calling an overridable method on each game object. Why is that?
And is that the only approach out there? Isn't it odd that a very objected-oriented approach is the default
in an industry that [keeps complaining that OOP is bad and we should do data driven instead](https://www.youtube.com/watch?v=rX0ItVEVjHc)?

In this series of articles we talk game simulation performance and architecture. In the
[previous episode](/2025/04/23/making_games_tick_part1/) we went over the basics of a game main loop and
how it is often implemented (although in a simplified manner for the sake of getting started). Today
we take a step back and think about flow and architecture.

## Why so abstract?

When I first had a look at commonly used engines (such as Unreal, Unity and Godot) and the way they handle ticks
it felt pretty weird and alien to me. I have spent most of my time in game development career working with custom engines,
especially with the Clausewitz Engine at Paradox Development Studio (don't bother looking it up on the internet,
most pages I could find about it are out of date, completely wrong, or both). Let me explain what I mean by bringing back
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
and then another guess about which ones are active and which ones may not.

A lot of criticism of Object Oriented Programming has been written over the years, especially in games
programming circles, so it is a bit curious to see at the heart of most game engines in its most canonically
wrong form.

This isn't an exaggeration, one of the biggest issues brought up against OOP architectures is
how they tend to overgeneralize concepts to the point of meaninglessness. Take a `Shape` or a `Mesh`
base class for example, they could have hundreds of derived implementations with their own specificities,
but you could reasonably make an argument for a common virtual `draw()` method. This is a fairly specific
task to be delegated to a virtual method which encapsulates the implementation details of how that shape
or mesh is drawn. But in the case of a game object, a `tick()` method would encapsulate just about _any and all_
behaviour without telling anything about it.

Contrast it with this implementation:

```cpp
struct World
{
    Player player;
    std::vector<Ghost> ghosts;
    std::vector<2DPoint> dots;
    std::vector<2DPoint> powerUps;
};

static World world;

void updateSimulation()
{
    movePlayer( world.player );
    eat( world.player, world.dots, world.powerUps );
    moveGhosts( world.ghosts );
    handleCollisions( world.player, world.ghosts );
    if ( world.player.dead )
    {
        gameOver();
    }
    else if ( world.dots.empty() )
    {
        loadNextLevel( world );
    }
}
```

While this isn't a particularly amazing implementation (you would probably want to abstract at least the part
that handles what happens when the player collides with a dot, power up or ghost), one could tell at a glance
that our game is probably some sort of Pac-Man clone and build a mental model of how the game simulation flows.
Most importantly (in my opinion), it expresses the simulation in terms of a series of tasks (do this, then do that...)
and their arguments (which game objects they might read or write to) from the top-down, instead of asking each
object to do its thing, and maybe impact the rest of the simulation in the process (bottom-up).

To some degree, this all goes back to the basic (but hard) software engineering principle of dividing the program
into sub routines and _giving them good names_. Having a virtual `Tick()` (or `Update()`, or `_process()`) achieves
some of the former, but definitely not the latter.

Lastly, having a bottom-up model makes it real hard to parallelize / use multiple threads to update the simulation,
as it makes it almost impossible to reason about data races. We will come back to this point in later episodes.

## The case for ticking

So, if this model is so bad, why is it so widespread? While I cannot claim to speak for the whole industry
(or even for the maintainers/architects of publicly available engines), I can probably make a few educated
guesses.

The most classic answer would be legacy. We do things this way because engines did it in the late 90s and early
2000s. If you take a look at the source code of IDTech based games of the era (such as
[Quake III Arena](https://github.com/id-Software/Quake-III-Arena/blob/master/code/game/g_main.c#L1713) or
[Half-Life](https://github.com/ValveSoftware/halflife/blob/master/engine/eiface.h#L432)) you will find a
`think` function pointer on each entity that is called each frame (with a timer check for cool down).

Game PCs had only one core up to early 2010s, so multithreading game simulation would bring more complexity
for little to no benefits. And the number of entities that needed to tick each frame was fairly low. If you add
up all the monsters, players, vehicles and projectiles (usually not including firearms, those mostly used hitscan)
at any given time in a level, you'd be in the low hundreds at most. And a significant portion of them wouldn't
need to tick each frame.

One important thing to note: all those engines (IDTech, Unreal, Source...) were originally made with first person shooters
in mind. As the old adage goes: use the right tool for the right job. When making a game that relies on a fairly complex
simulation of the world such as Grand Strategy Game, with intrinsics hierarchies and lots of interactions between game objects
outside of physics, I would argue it is important to be able to reason about the big picture. When making a first or
third person game, where a lot if it comes down to physics simulation, which is usually handled as its own separate thing outside of
object ticks (unless a game decides to entirely ignore the engine's physics system and implements its own), maybe less so. I do however
suspect this model would become unwieldy and hard to optimize when used for game AI at scale. For example try to imagine how you would simulate a 
large number of NPCs in a busy city going about their day around the player.

## Ticking's final boss

Finally, there is one big challenge when it comes to reasoning about a tick's architecture and that's external bindings.
Or as they most commonly manifest: scripting. How can one quickly assess what a game object tick does if it's not
part of their Visual Studio solution and they need to find/open an external text file written in another language (or worse, a visual script)?
Short answer: they can't, at least not easily or without good tooling.

For the longer answer, we will have to wait for part 3 of this series. See you next time!
