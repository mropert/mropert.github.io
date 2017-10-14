---
layout: post
title: My CppCon 2017 talk about Modular Design with CMake is online!
tags: [cpp, events]
description: >
  Do you think about your libraries in CMake in term of compiler flags or architecture modules? In this talk, I show you how to do
  the latter.
author: mropert
---

For those who couldn't make it to CppCon 2017, the video of my first talk "Using Modern CMake Patterns to Enforce a Good Modular Design"
is now online:

<iframe width="560" height="315" src="https://www.youtube.com/embed/eC9-iRN2b04" frameborder="0" allowfullscreen></iframe>

You can also find the slides [here](https://github.com/CppCon/CppCon2017/blob/master/Tutorials/Using%20Modern%20CMake%20Patterns%20to%20Enforce%20a%20Good%20Modular%20Design/Using%20Modern%20CMake%20Patterns%20to%20Enforce%20a%20Good%20Modular%20Design%20-%20Mathieu%20Ropert%20-%20CppCon%202017.pdf).

### Background

When I started using CMake for a big project at work, I was a bit disappointed by the number of resources on best practices.
Granted the [CMake documentation](https://cmake.org/cmake/help/v3.10/) is not nearly as bad as people make it, but I'll admit
it's not really friendly looking either.

The best I could find at the time was Rix0r's blog post: https://rix0r.nl/blog/2015/08/13/cmake-guide/.

Since C++Now this year, you can also see [Daniel Pfeifer's Effective CMake](https://www.youtube.com/watch?v=bsXLMQ6WgIk), which gives
a very good overview of what CMake can do today.

There were many things I could have talked about, so in the end I focused on Modular Design.
It's a notion I took from John Lakos' book [Large-scale C++ Software Design](https://books.google.fr/books?id=AuMpAQAAMAAJ),
his [talk](https://www.youtube.com/watch?v=QjFpKJ8Xx78&index=99&list=PLHTh1InhhwT7J5jl4vAhO1WvGHUUFgUQH) at CppCon 2016
and David Sankel's talk on [Building Software Capital](https://www.youtube.com/watch?v=ta3S8CRN2TM).

The idea is that we as software engineers need to keep control of our code architecture or risk falling into the circular-dependency circle
of hell where nothing works in isolation, which makes it impossible to reuse or unit test. The bigger the project, the higher the chance.

If there is one place where it should be easy to see the code architecture of a project, it should be its build files.
After all, this is where we are expected to describe the links between the various parts.
And unfortunately, the "traditional" way of building software we inherited from `make` doesn't help at all.
Instead of telling "that stuff depends on this stuff", we write a lengthy script that invokes the compiler as many times as needed
(and some more, just to be on the safe side).

More recent attempts, such as Bazel and its derivatives, have favored description over scripting. What I discovered is that CMake
is also able to do this. You won't get rid of scripting, but you can at least describe your libraries in terms of dependency graphs
instead of `-Ithat_lib` flags.

So go watch it if you haven't already and tell me what you think.

PS: And now I realize, there's no way to comment here. I'll have to work on that. In the meantime, there's always Twitter, e-mail
and the cpplang chat.
