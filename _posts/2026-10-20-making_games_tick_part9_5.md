---
layout: post
title: "What makes a game tick? Part 9.5 - The Curse of the Scheduler Latency"
tags: [cpp, gamedev]
description: > 
  Let's talk about game simulations. Today I bring up a slightly more advanced topic
  and talk about OS scheduler latency and its impact on multi-threaded tick design.
author: mropert
---

After [part 9](2026/02/27/making_games_tick_part9/) we were done with the thread scheduling part
of our data-driven tick system and getting ready to move on to the data storage part of the problem.
But in the interval I got to do some performance audits on mobile and ran into some interesting results
that I feel I should share before we go on.

## Scheduling tick tasks

In the past articles we focused on how to turn our list of tick updates into a task graph,
but we did not go into the specifics of how to schedule and execute them.

This was a conscious choice from me, as the details of a task scheduler would be out of scope for this
article series. Besides, there are already good parallel execution libraries out there that would do a
good job<sup>[1](#myfootnote1)</sup>, and chances are your game engine also comes with a perfectly
serviceable task system.

Assuming your task system has a `scheduler.enqueue()` and a `scheduler.wait_all()` function, running the
graph would roughly equate to:
1. Set an atomic wait count on each task equal to its predecessor count in the graph
2. Wrap the task body callback so that before returning it atomically decrements the wait count
    for all successor tasks, and enqueue the ones where the count is decremented to 0
3. Enqueue all the tasks where wait count is 0 and call `scheduler.wait_all()`

There are some extra optimizations you can add on top, but for the purpose of our series this will suffice.

## Execution and yielding

Task schedulers work by having each work thread in the pool wait on the task queue not being empty,
then popping the first task in the queue, running it, then looping again.

While textbook runners will implement this with a simple `cv.wait( mtx, [&] { return _queue.size() > 0; }`,
production task schedulers usually implement a back-off mechanism where they spin for a while before sleeping
on the "work available" condition variable.

This is because [`cv.wait()`](https://en.cppreference.com/cpp/thread/condition_variable/wait) is a blocking operation
that yields back the CPU to the OS scheduler until the condition variable is signaled (usually by the
`scheduler.enqueue()` function calling `cv.notify_one()`), requiring two or more thread context switches.

For real-time applications (like games), it's usually better to try to keep the CPU spinning for a bit
in case more work shows up. On Windows, this amounts to a 4 to 15 microseconds latency<sup>[2](#myfootnote2)</sup>
between one thread issuing a notify and a waiting thread resuming execution.

So how long should a worker thread spin before sleeping? It depends.

As often in computer science, it all comes down to a tradeoff. A longer ramp-down period will increase
responsiveness if we queue a mix of parallel and serial jobs in succession, at the cost of more CPU usage
on idle and oversubscription.

From my practical experience on optimizing ticks on desktop games, this is usually already well handed by your
threading library. TBB will execute up to 64k `pause` instructions before yielding to the scheduler, and then finally
sleeping. Unreal has a similar mechanic but counts machine cycles rather than relying on the machine-dependent timing of
the `pause` instruction. I've seen custom engines use real time delays up to 1ms before sleeping because their tick relied
a lot on unsorted serial/parallel passes.

## What about mobile?

That's all well and good on desktop because the scheduler latency is still manageable, and worse case the
downside of aggressive spinning is usually acceptable. It's mostly annoying to players who keep a video
playing on their second monitor, and to some streamers<sup>[3](#myfootnote3)</sup>. But what about mobile?

In my experience, the iOS scheduler behaves similarly to desktop. But on Android, I found out that
the worker thread wake-up delay was prohibitive for cases that worked fine on desktop and iOS.
This is due to a mix of things, but I observed latencies that were 10 times worse (in the hundreds of microseconds).

Sadly, I cannot share the captures, but I observed two usage classes that went awry:
1. Physics simulation, especially small queries that created "island chains"
2. Entities tick mixing up a lot of serial work with parallel jobs passes

## Hiding latency with work stealing

When dealing with worker thread pools, ideally the scheduling thread would try to steal jobs rather than
issue a blocking wait on the job list, which would handle small async jobs: they would all
be stolen and completed on the scheduling thread if the worker threads can't wake up fast enough.

This is the default on TBB, a call to `tbb::task_group::wait()` will have the calling thread join
the worker pool and try to steal some work and handle it. I'd argue stealing is the right default,
but there are some edge cases to be aware of.

The main one being that the scheduler must ensure proper isolation so that the main thread only cares
to steal jobs that it would otherwise be waiting on. TBB handles this by having task groups that
worker threads from the main pool join and leave as the work ramps up and down. However, if the caller
does not explicitly request [work isolation](https://uxlfoundation.github.io/oneTBB/main/tbb_userguide/work_isolation.html),
a blocking `wait()` call may end up a stealing task from a different group. In the case of a game, that could for example
mean that waiting on work threads to complete a small physics job could lead to the main thread grabbing
a shader compile job and add significant latency to the critical path.

Depending on the way our task system works and how our physics library is integrated, it is possible
that implementers would elect to put a basic blocking wait instead of risking work stealing going
rogue.

Another issue is that even if we implement proper work stealing, our main thread still needs to
wake the worker threads through `notify_one()` or `notify_all()`. That means jumping into kernel space,
selecting which threads to mark as ready and which CPUs to wake up with IPIs. The
[previously mentioned](https://alexanderhoughton.co.uk/blog/comparing-synchronisation-methods-on-windows/)
showed that the wake system call cost about as much latency (if not more) as the worker thread wake up.

This means even if you hide the wake-up latency with work stealing, you are still losing some amount
of time simply notifying other threads. And for tiny jobs like our physics case above, you likely
end up paying more to schedule an async job than you would by executing it directly without calling
any threading primitives.

In the mobile game where I measured it, I could immediately gain 1ms of frame time on Android by disabling worker
threads entirely in Unity. Obviously that's not an amazing solution because other systems (like scene culling)
will genuinely need them. I believe a setting will be added in the future to disable multithreaded physics,
although I don't know in which release.

## Lessons for our tick

There are three things to consider from what we learned:

One, work threads have a spin-up latency, which can be mitigated by keeping them busy. This means trying to group
parallel work close together and avoiding sync points as well as alternating serial/parallel passes.

Two, signalling work threads that work is available has a cost for the calling thread itself, so we should measure
before turning `for()` into a `parallel_for()`.

Three, the sweet spot to alleviate those first two issues depends on platform. On PC, Mac and iOS signaling and waking
up threads both cost about 5-15 us. So our cutoff for parallel work should be about double that number. On Android,
that number is much higher (up to 10 times) and our heuristic/rule of a thumb should factor that.

And with that, we should be able to get back to our scheduled (pun intended) program in the next post, for real this time.

---

<a name="myfootnote1"><sup>1</sup></a>: I'd still recommend Intel's
[Thread Building Block](https://github.com/uxlfoundation/onetbb) today.

<a name="myfootnote2"><sup>2</sup></a>: The number initially came from a 
[StackOverflow thread](https://stackoverflow.com/questions/70355069/synchronization-primitives-equal-latencies-between-atomics-and-mutex-lock),
but got recently by a [benchmark by Alexander Houghton](https://alexanderhoughton.co.uk/blog/comparing-synchronisation-methods-on-windows/).
Turns out they might have been on the conservative low-end estimate.

<a name="myfootnote3"><sup>3</sup></a>: The professional streaming studios I've seen during my career used a second computer
with a capture card for encoding to avoid any hitch caused by the game CPU usage.

