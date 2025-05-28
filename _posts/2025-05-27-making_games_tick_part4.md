---
layout: post
title: "What makes a game tick? Part 4"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today we look at some real life script performance issues I had to fix.
author: mropert
---

In [the previous article](/2025/05/07/making_games_tick_part3/) I brought up the idea that scripting in games can
often be a source of performance bottleneck because it's a powerful code-like tool used by people who are usually
not trained to be programmers. And to be honest I expected to get more flak for it, but to my surprise the comments
were mostly pointing out games that figured out how to restrict scripting for better performance (although interestingly
all of them were using bespoke engines rather the most commonly licensed ones). So I figured this week let's look
at some "bad" script and discuss how we got there. My hope is that this will give both my designer and programmer
readers some insights.

Those examples will be inspired by real issues I had to fix in Paradox games, and if you're curious you can probably
find the originals in the game files in your Steam folder (unless you bought the games through the Microsoft Store but
why on Earth would you do that?!).

## Finding Bulgaria the hard way

Hearts of Iron IV is a game about World War 2 on which I was tech lead for several years. Each country in the world
is represented by an entity, and each of these entities can (among many, many other things) make a political decision
from an array of available options that depend on the state of the game. Designers add new decisions all the time
to bring more flavour and content to the game. Here's one if you are playing Bulgaria and want to offer your ally
a concession to your uranium deposits to help them in their atomic research program:

```lua
target_trigger = {
	FROM = {
        is_in_faction_with = ROOT
        OR = {
            is_faction_leader = yes
            AND = {
                is_major = yes
                BUL = { is_faction_leader = yes }
            }
        }
    }
    controls_state = 48
	has_completed_focus = BUL_uranium_prospecting
}
```

I assume most of my readers aren't familiar with the beauty of PDS script so let me walk you through it. This `target_trigger`
is a condition that must be true in order for a decision to be available and shown in the country political decisions UI.
Each in-game day each country entity will check this trigger against _every other country entity_ to see if this is true.
Next up we have 3 checks that are made.
1. The `FROM` block is a bit verbose but what it boils too is checking "am I in the same faction as the other country" and "is either of us the leader of the faction or a major power".
2. Am I in control of state #48 (the Sofia/Vidin region)
3. Did I complete the national focus `BUL_uranium_prospecting` which boils down to did I, _as Bulgaria_ play through the bit
of content that opens up uranium processing.

Do you see the issue yet? Let's write some pseudo C++ equivalent:

```cpp
for ( const auto& ourCountry : AllCountries )
{
    for ( const auto& theirCountry : AllCountries )
    {
        if ( ourCountry != theirCountry && ourCountry.getFaction() == theirCountry.getFaction()
            && ( ourCountry.isFactionLeader() || theirCountry.isFactionLeader() )
            && ourCountry.controlsState( 48 ) && ourCountry.hasCompletedFocus( "BUL_uranium_processing" ) )
        {
            // Decision available, add it to the UI...
        }
    }
}
```

This way makes it more obvious that is a quadratic algorithm (scales to the number of countries squared) and it doesn't need to be.
In this implementation each country checks against every other country for a condition that will always be true or false (am I Bulgaria,
did I complete the uranium processing content and do I control Sofia). This is a common loop/condition mistake that most software engineers
should encounter (and learn not to repeat) in university. A good static analyzer might even be able to find it
(although given how much legacy C++ abuses side effects and `const_cast` it may raise false positives).

In this C++ version it would be easy enough to fix, a simple refactoring like this should do the trick:

```cpp
for ( const auto& ourCountry : AllCountries )
{
    if ( ourCountry.controlsState( 48 ) && ourCountry.hasCompletedFocus( "BUL_uranium_processing" ) )
    {
        for ( const auto& theirCountry : AllCountries )
        {
            if ( ourCountry != theirCountry && ourCountry.getFaction() == theirCountry.getFaction()
                && ( ourCountry.isFactionLeader() || theirCountry.isFactionLeader() ) )
            {
                // Decision available, add it to the UI...
            }
        }
    }
}
```

We could add a `break` at the end of the `if` block since this condition can only be fulfilled by one country at
a time, but this wouldn't impact performance nearly as much as adding the outer `if`. And also I'm not a big fan of `break`
(or `continue`).

## Fixing the script

Now obviously the original code wasn't in C++, it was part of the game script. And the way it is currently exposed
to the designers they can only write a condition that will be checked in the inner loop. Addressing this would require
scripters to be able to add a second trigger that is only checked once per country, before the inner loop is executed.
This is how we fixed it back then, we added an optional script block for every decision called `target_root_trigger`
which bypasses the entire inner loop when false.

This illustrates a point about script bindings: they need to balance convenience and performance. Consider this "fixed"
version of the original script:

```lua
target_trigger = {
	FROM = {
        is_in_faction_with = ROOT
        OR = {
            is_faction_leader = yes
            AND = {
                is_major = yes
                BUL = { is_faction_leader = yes }
            }
        }
    }
}

target_root_trigger = {
    controls_state = 48
	has_completed_focus = BUL_uranium_prospecting
}
```

Is it faster? Yes. Very much so. But now we have to teach every designer about `target_trigger` vs `target_root_trigger` and that
may become unwieldy. The current version on Hearts of Iron IV has _six_ different triggers for decisions and half of them only exist filter out
the space of script triggers that need to be evaluated. They do not alter the game behaviour, they are just performance
hints to the script system.

## Going further

Now could this be automated? In theory yes. With enough metadata the system could classify condition statements by which data
they depend on, and then only run them when necessary.

This would require the ability to split the AST ([Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree))
that represents each of these `target_trigger` by extracting the statements that only depend on the outer loop country and executing these
outside the inner loop. In the above case that shouldn't be too difficult because we clearly see the bits in the `FROM` block are
using the inner loop variable (in this example `FROM` is the inner country, `ROOT` in the inner country, explaining
why the scripting language was made that way would take several posts and I'm not even sure I know all the answers). We would
also need to check for expressions where `FROM` is used on the right hand side (in this case there are none).

Assuming we have a way to unit test our script engine, we could introduce that automated optimization with a degree of confidence.
And of course since a lot of the development cost of this feature isn't really tied to this particular script/code boundary, we
should consider applying it to all the places where we run a lot of script triggers in a loop. This would have a big upfront cost
but would likely pay off in the long run as this would not only speed up the game but also simplify the script system for
designers.

## Parting words

I initially planned to include more examples in this post but we are already reaching my target size so I'll keep that for the
next one so this series stays regular and digestible. Next time we will look at another real life performance issue, this time
from art script.

Another reason this post took a while is that I've started a [consulting](/consulting) activity. Should you be in need of some
expertise for performance programming or training in C++ feel free to reach out!
