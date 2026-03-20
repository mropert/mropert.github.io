---
layout: post
title: "Looking at Unity finally made me understand the point of C++ coroutines"
tags: [cpp, gamedev]
description: > 
  I had seen many talks about coroutines but it never really clicked where I could use them
  outisde of async IO. Until I looked at how Unity uses them in C#.
author: mropert
---

Coroutines have been around in C++ for 6 years now. And still I have yet to encounter any in production code.
This is possibly due to the fact that they are by themselves a quite low-level feature. Or more precisely,
they're a high level feature that requires a bunch of complex (and bespoke) low-level code to plug
into a project. But I suspect another, even bigger, issue with the coroutines rollout in C++ has been the
lack of concrete examples. After all, how often do you need to compute Fibonacci in real life?

Recently, I have been looking at Unity, which mostly uses C# for client gameplay code (you can do C++ but it's uncommon).
And more specifically, I ran across their usage of coroutines for spawning effects and other ephemeral behaviours.
Here's an [example from the manual](https://docs.unity3d.com/6000.3/Documentation/Manual/Coroutines.html) I'll reproduce here
for the purpose of illustrating this article:

```cs
void Update()
{
    if (Input.GetKeyDown("f"))
    {
        StartCoroutine(Fade());
    }
}

IEnumerator Fade()
{
    Color c = renderer.material.color;
    for (float alpha = 1f; alpha >= 0; alpha -= 0.1f)
    {
        c.a = alpha;
        renderer.material.color = c;
        yield return null;
    }
}
```

C# and/or coroutines purists might take offense at this usage of `yield`. After all the semantics are all wrong here. We're yielding nothing
where we're trying to express something akin to `await NextFrame()`. From what I could read this is an artifact inherited from a lack
of `await` support when they were initially added to C# (they only supported generator style `yield`), which led Unity to use this hack which
is still around today. I am not only mentioning it as a random piece of historical trivia, this will become relevant later.

## Why coroutines?

This example is still a bit basic and might not make it immediately apparent why we would prefer to write our effects this way.
After all, this could be made into a simple lambda with a mutable `alpha` variable that we would nudge each call.
But let's try with a slightly more complex effect:

```cs
IEnumerator TimeWarp()
{
    // It's just a jump to the left
    transform.position.x -= 1.f;
    yield return null;

    // Then a step to the right
    for (int i = 0; i < 4; ++i)
    {
        transform.position.x += 0.2f;
        yield return null;
    }

    // Put your hands on your hips
    // ...

    // Let's do the time warp again!
    for (int i = 0; i < 4; ++i)
    {
        transform.Rotate(0.f, 90.f * i, 0.f);
        yield return null;
    }
}
```

Now it would become actually painful to turn this into a regular functor or lambda. Writing it in C++ turns it into
some sort of ugly state machine like this:

```cpp
class TimeWarp
{
    enum class State
    {
        Jump,
        StepRight,
        HandsOnHips,
        // ...
        DoAgain
    };

    State _state = State::Jump;
    int _i = 0;
    Transform* _transform;

    TimeWarp(Transform& transform) : _transform(&transform) {}

    bool operator()()
    {
        switch ( _state )
        {
            case State::Jump:
                _transform->position.x -= 1.f;
                _state = State::StepRight;
                break;

            case State::StepRight:
                _transform->position.x += 0.2f;
                if ( ++_i == 4 )
                {
                    _state = State::HandsOnHips;
                    _i = 0;
                }
                break;

            // ...

            case State::DoAgain:
                _transform->Rotate(0.f, 90.f * i, 0.f);
                if ( ++_i == 4 )
                {
                    // Indicate we're done
                    return true;
                }
                break;
        }
        return false;
    }
}
```

Pretty ugly, isn't it? Would you let it pass code review? What else would you suggest instead?

I guess I would perhaps recommend the author split `TimeWarp` into its component moves and handle state
transitions by queueing the next effect as a continuation. But I probably wouldn't be happy about it.

This, to me, is the kind of no-brainer case I've been dying to see to be sold on the value of coroutines.
Wrapping one loop might not be worth the hassle of figuring out how to integrate coroutines in your codebase, but
wrapping a sequence of operations with state definitely does. It's all about turning a hard to read state machine into
a very simple function.

## A C++23 implementation

So, let's do the time warp again in C++ then.

```cpp
std::generator<std::monostate> TimeWarp(GameObject& obj)
{
    // It's just a jump to the left
    obj.transform.position.x -= 1.f;
    co_yield {};

    // Then a step to the right
    for (int i = 0; i < 4; ++i)
    {
        obj.transform.position.x += 0.2f;
        co_yield {};
    }

    // Put your hands on your hips
    // ...

    // Let's do the time warp again!
    for (int i = 0; i < 4; ++i)
    {
        obj.transform.Rotate(0.f, 90.f * i, 0.f);
        co_yield {};
    }
}
```

My readers may object that this is a hack. In fact, this is the same hack as Unity did back a decade and some change ago.
And that's precisely the point. For the exact same reasons.

See, the real reason we mostly see Fibonacci generators in slides is because using `co_yield` is (relatively) easy, especially
since C++23 gave us `<generator>`. But making use of `co_await` is _hard_. Yielding from a coroutine is fairly straightforward and generic.
The control flow is simple, we suspend and return to the caller and _they_ decide when we will be awaken next. On the other
hand handling `co_await` requires answering _a lot_ of questions that don't have an obvious answer. What are we going to
wait on? How will they signal that they are ready to resume? Can we use signals/interrupts instead of polling? Who will check that they
are ready to run again? Will they also awaken (run) the coroutine, or will they put them back in an execution queue?
Which execution queue? A background thread? A thread pool? Using what implementation? The list goes on.

To misquote Kennedy, "we chose to focus coroutines on `generator` in C++23, not because it is hard, but because it is easy".

C++26 should implement [`execution`](https://wg21.link/p2300) and give us a framework to be able to use `co_await`, but I expect
it to be an uphill battle. After all, most projects should already have their own concurrency solution and given how little
is in the standard besides low level constructs, it means a lot of divergence that will need to be plugged back into the `execution` model.
I expect most projects have their own custom schedulers, thread pools and the like. Or use something like TBB to get one.

Perhaps your codebase already uses `boost::asio` in which case you
[already have support for coroutines](https://www.boost.org/doc/libs/latest/doc/html/boost_asio/overview/composition/cpp20_coroutines.html).
If not, you will either need to wait for C++26 and switch/integrate with `execution`, or implement your own `promise`s and awaitables
to fit your threading model.

Or you could use the Unity hack.

## Unity-like coroutines runner in C++

It took me less than an hour to implement a simple Unity style coroutine executor in my toy game main thread.
Here's the whole thing:

```cpp
class effects_manager
{
public:
    void add( std::generator<std::monostate> effect )
    {
        _effects.push_back( std::move( effect ) );
        _iterators.push_back( _effects.back().begin() );
    }

    void run()
    {
        // Remove the ones that are done (tweaked https://en.cppreference.com/w/cpp/algorithm/remove.html#Version_3)
        int first = 0;
        for ( ; first != _effects.size() && _iterators[ first ] != _effects[ first ].end(); ++first )
            ;
        if ( first != _effects.size() )
        {
            for ( int i = first; ++i != _effects.size(); )
            {
                if ( _iterators[ i ] != _effects[ i ].end() )
                {
                    _effects[ first ] = std::move( _effects[ i ] );
                    _iterators[ first ] = std::move( _iterators[ i ] );
                    ++first;
                }
            }
            _effects.erase( begin( _effects ) + first, end( _effects ) );
            _iterators.erase( begin( _iterators ) + first, end( _iterators ) );
        }

        // Run the effects
        for ( int i = 0; i < _effects.size(); ++i )
        {
            ++_iterators[ i ];
        }
    }

private:
    std::vector<std::generator<std::monostate>> _effects;
    using effect_iterator = decltype( std::declval<std::generator<std::monostate>>().begin() );
    std::vector<effect_iterator> _iterators;
};
```

That's it. The only hard part is the loop that removes the coroutines that have reached the end of their execution
by hand-writing a `std::remove_if` variant that works with 2 zipped arrays. If you already have a utility for it,
the whole thing will take less than 20 lines.

Now can fire effects by writing something like `effects.add(TimeWarp(object))` and we just need to remember to call
`effects.run()` in our main loop.

Doing it the "proper" way would require to write a custom next-frame awaiter that inserts our coroutine handle into
a next frame queue. While that's doable, this requires a more in-depth understanding of coroutines internals to implement.
And, to be honest, I kind of like the `yield` approach to mean "yield control until next frame".

## Bonus benefit

As I was writing this, I also realized, it wouldn't take much to turn our current implementation into a proper generator
rather than relying on our coroutine invoking side effects. Instead of `monostate` we could return a renderable object.

```cpp
std::generator<Draw> TimeWarp(const Model& model)
{
    // It's just a jump to the left
    vec3 position{ -1.f, 0.f, 0.f };
    co_yield Draw{ .model = model, .transform{ .position = position } };

    // Then a step to the right
    for (int i = 0; i < 4; ++i)
    {
        position.x += 0.2f;
        co_yield Draw{ .model = model, .transform{ .position = position } };
    }

    // Put your hands on your hips
    // ...

    // Let's do the time warp again!
    for (int i = 0; i < 4; ++i)
    {
        obj.transform.Rotate(0.f, 90.f * i, 0.f);
        co_yield Draw{ .model = model,
                       .transform{ .position = position,
                                   .rotation = Rotate(0.f, 90.f * i, 0.f) } };
    }
}
```

Now we change our `run()` method to populate a vector of draws:

```cpp
std::vector<Draw> run()
{
    // Remove the ones that are done ()
    // ...

    // Run the effects
    std::vector<Draw> draws;
    draws.reserve( _effects.size() );
    for ( int i = 0; i < _effects.size(); ++i )
    {
        draws.push_back( *_iterators[ i ] );
        ++_iterators[ i ];
    }
    return draws;
}
```

And while we're at it, we could even make our loop run in parallel now since we removed the side effects:

```cpp
// Run the effects
std::vector<Draw> draws( _effects.size() );
tbb::parallel_for( 0zu, _effects.size(), [this, &draws]( size_t i )
                   {
                       draws[ i ] = *_iterators[ i ];
                       ++_iterators[ i ];
                   } );
return draws;
```

There. A simple and relatively efficient effect system for our game that allows designers to implement all sorts
of bespoke funky things as easy to read coroutines, and the entire system took us less than a hundred lines to write.

Now, wouldn't you say this looks much more interesting to have than if I had shown you yet another Fibonacci generator?
