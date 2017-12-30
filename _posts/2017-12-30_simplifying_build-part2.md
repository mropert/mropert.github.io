---
layout: post
title: Simplifying build in C++ (part 2)
tags: [build, cpp]
description: >
  Package management and build systems are one of the next big challenges C++ is going to face.
  In this article series, I offer some thoughts and ideas to solve that.
  This post is the second part, where we talk about toolchain definition, how a standard could work and what challenges it
  would face.
author: mropert
---

Christmas and New Year Eve are that particular parts of the year when we wish for impossible things and make unreasonable promises.
Get a bottle of a 1999 DRC La Tâche, lose weight, have a C++ ecosystem with a built-in package manager...

Since we computer scientists like challenge, let's take a moment to work on the hardest of those. In the 
[previous](/2017/10/19/simplifying_build-part1/) article, we layed out a basic design for a package management standard
in C++. It is not perfect, but as I explained in the [quick follow-up](/2017/10/24/simplifying-build-and-api-convergence/),
perfection was not he primary goal. That spot was reserved for compatibility and migration paths from the existing C++ ecosystem.

With that thought in mind, let's explore one bit of the design more in detail: the toolchain configuration.

### Toolchain configuration and why you should have one

Simply put, the toolchain is the compiler configuration. It tells the build system which compiler it should use, with
what options. It's usually not longer than half a dozen lines, but each has tremendous impact on the result, determining
which machine will be able to run the resulting binaries, how fast it may perform, if it will be possible to debug it or not
and what other binaries it may be compatible with.

As odd as it may appear, defining a toolchain is not mandatory on all build systems (or package managers). For example
if you lookup [CMake's documentation](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html), it will describe
it as something mostly reserved for cross-compiling. At first it may seem reasonable, CMake is perfectly capable of detecting
whatever machine and compiler you have and make the decision for you. But the second you start sharing your code with someone
else, or distribute binaries, the whole "on my machine, it works" falls apart. You need to define a common baseline.

In essence, a toolchain configuration is used to describe two things:
* What the code requires to compile (host requirements)
* What machine will be needed to run the produced binaries (target requirements)

The distinction between the two may seem obvious if you've worked on cross-compiled projects, but I've met a lot people who
did not exactly thought them as separate. Perhaps their build infrastructure was always a 100% match with the target production
servers, or simply they never add the "luck" of hitting one of the many small issue that can arise from it. Here are some
examples:

* Compiler optimizations may use CPU features that are available on the host but not the target
* Runtime libraries (libc, libstdc++...) may differ between host and target, resulting in potentially incompatible binaries
* Debug binaries cannot be distributed outside the build/test machines on some platforms (Windows MSVC for example)



--- TODO

* Compiling with either C++03 or C++11 may seem like a host-only requirement (does the compiler support C++11 or not), but in fact
may impose restriction on the target as well (binaries for C++03 libstdc++ are not ABI compatible with binaries for C++11).
* 

