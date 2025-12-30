---
layout: post
title: "A Year With Graphics"
tags: [cpp, gamedev, graphics]
description: > 
  Looking back at 2025 and looking forward to 2026 through the lens of graphics programming.
author: mropert
---

I had done some work with graphics while working on various titles at Paradox, but I never felt
really confident about it like I would have been about C++ or multithreading or the few other
topics I've talked about in the past. Sure I had done some work with it, figured out what the point
of shaders is (the answer is: they shade) and migrated Hearts of Iron IV from DirectX 9 to 11,
but it still felt a bit mystical. So I decided to use my spare time between contracts this year
to catch up.

## If at first you don't succeed...

This wasn't the first attempt I had made. A few years back I had run across a series of articles
online claiming to make it "easy to understand", only to welcome the reader with thousands of
lines of Vulkan bootstrap code (all in C, of course). I found the whole thing utterly impossible
to digest and moved on with my life.

This year, I started again with [raylib](https://www.raylib.com/) after a suggestion from a past coworker.
I had more experience with DirectX than OpenGL, but in combination with the [Learn OpenGL](https://learnopengl.com/)
this proved easy enough to catch-up. I combined it with the [C++ wrapper](https://github.com/RobLoach/raylib-cpp)
to avoid manual resource management because that's definitely not a thing we should be doing
30 years after RAII was invented.

Running into some limitations, I then gave a shot to SDL3_GPU after seeing a 
[presentation from Mike Shah](https://www.youtube.com/watch?v=XHWZyZyj7vA). The main value
I found in it was making it obvious that one needs to bulk buffer updates
in big batches because doing small `mmap()`/`memcpy()`/`munmap()` is really 
[really inefficient for GPUs](https://developer.nvidia.com/content/constant-buffers-without-constant-pain-0).
In exchange however I'd lost all the other features brought by raylib, including the math library
and asset loading.

Sadly (and despite being officially released in 2025), SDL3_GPU still enforces antiquated
patterns like vertex buffer layout description and other fixed function pipeline relics. In general
the API lacks support for bindless resources and it's unclear when (or if) 
[it will be added](https://github.com/libsdl-org/SDL/issues/11148).

And so after all those adventures I was back on Vulkan. And this time it feels like I succeeded.

## Learning

This whole process brought me back to a topic that is dear to me: learning. More precisely,
how does one get into something new without easy access to experts that can point you in the right
direction? After years in the C++ conference circuit I had kind of taken for granted that there
was always someone a DM away from the answer, or at least a good lead towards it.

There's a lot of stuff out there, so how does one find the right resource? Assuming we can at
least avoid nonsense written by AI (hint: you can ask google for pages published before 2023),
that's still a big haystack. One of the hardest part, I found, was to figure out what was
the latest trends and best practices. C++ is not the only tech topic guilty of using the word
"modern" to describe patterns that are now a decade old...

One of the things that helped were the presentations made at [ACM SIGGRAPH](https://www.siggraph.org/).
While far from perfect (finding anything on their website is near impossible and reposts on Youtube seem
to happen on a random schedule) and often hard to get into as beginner, the slides did come with
neat bibliographies which proved very useful to vet sources and articles. If it's cited in
a recent presentation about IDTech or Frostbite, it's probably solid.

Eventually I found out about the [Rendering Engine Architecture Conference](https://enginearchitecture.org/index.htm)
which talks are consistently uploaded on Youtube. I don't know the whole story behind it
(they claim to be "a reaction to the conferences they used to attend") but after the
hurdles of accessing SIGGRAPH (to say nothing about GDC) I certainly think they might be on to something.

## Results

My last rewrite started by following up the (unofficial) "[Vulkan Guide](https://vkguide.dev/)" which
proved useful to start up (although it's going through a rewrite and the last chapters are still missing),
which some extra inspiration found in a hobby project called [Kaleidoscope](https://github.com/Floating-Trees-Inc/Kaleidoscope)
that the social media algorithm randomly threw at me.

I put my own thin Vulkan abstraction out on [github](https://github.com/mropert/vk-renderer) although I don't
think there's much interesting stuff going on in there for now. The only feature I'd say is possibly
worth a look at is the [pipeline manager](https://github.com/mropert/vk-renderer/blob/main/src/renderer/pipeline_manager.cpp)
that implements background shader recompilation if the source changes. If you're curious about multi-threaded
asset loading in general, I made some experiment with various solutions [a month ago](/2025/11/21/trying_out_stdexec/).

In general there's a lot of stuff missing or possibly inefficient, as I try to only add features as I need them.
If decades of API development have taught me one thing, it's to never write something you don't have a client for.

Behold!

![Amazing GLTF asset renderer](/assets/img/posts/miku_vulkan.jpg)

If you bothered to check the repository, you might have noticed that I used C++20 modules. It only took
5 years, and I still needed to [hack around Intellisense](https://github.com/mropert/vk-renderer/blob/main/src/renderer/common.h#L9)
but I finally got to use modules. And yes it's an amazing quality of life for compile times when including C++ libraries.
When it works.

The other C++20 feature I can't live without now it's designated initializers. In my opinion they beat any form
of builder pattern or constructor overloads.

## What's next?

My initial plan was to implement mesh shading, but I was at first hesitant given the minimum hardware requirements
(RTX 2xxx series and later if I'm not mistaken). A [recent post](https://www.sebastianaaltonen.com/blog/no-graphics-api)
by Sebastian Aaltonen is starting to convince me that this is a reasonable baseline, and that my API isn't
going in a terrible direction. Phew!

My most recent watch has been the [Niagara series](https://github.com/zeux/niagara) by Arseny Kapoulkine which
I found very knowledgeable, but the streaming format can make the pacing a bit tedious at times. I wish
I could find a more edited video series. If anyone has any recommendation, I'm all ears. If not, this
might mean there a gap waiting to be filled.

Speaking of streaming-like content, I could not end this post without mentioning
[Freya Holmér's channel](https://www.youtube.com/@acegikmo) for anyone who would like a refresher on either
graphics math or shader basics. Again, please note this is also captured from a streaming format
and so the editing (or lack thereof) might annoy some.

## The search continues

As this year ends, I am left with an interesting question that has been a theme through this whole article:
have I managed to catch up? To answer the question requires not only looking at what I've learnt, but more
importantly figuring out what I don't know. And here lies the catch: you can never really know what you don't know.
At best you can do what I've done in this article: put something out there, and see if someone points you
at something you missed.

Happy new year!
