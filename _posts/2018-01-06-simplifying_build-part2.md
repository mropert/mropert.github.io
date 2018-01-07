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

Christmas and New Year Eve are that particular part of the year when we wish for impossible things and make unreasonable promises.
Get a bottle of a 1999 DRC La Tâche, lose weight, have a C++ ecosystem with a built-in package manager...

Since we computer scientists like challenge, let's take a moment to work on the hardest of those. In the 
[previous](/2017/10/19/simplifying_build-part1/) article, we laid out a basic design for a package management standard
in C++. It is not perfect, but as I explained in the [quick follow-up](/2017/10/24/simplifying-build-and-api-convergence/),
perfection was not the primary goal. That spot was reserved for compatibility and migration paths from the existing C++ ecosystem.

With that thought in mind, let's explore one bit of the design more in detail: the toolchain configuration.

### Toolchain configuration and why you should have one

Simply put, the toolchain is the compiler configuration. It tells the build system which compiler it should use, with
what options. It's usually not longer than half a dozen lines, but each has tremendous impact on the result, determining
which machine will be able to run the resulting binaries, how fast it may perform, if it will be possible to debug it or not
and what other binaries it may be compatible with.

As odd as it may appear, defining a toolchain is not mandatory on most build systems (or package managers). For example
if you lookup [CMake's documentation](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html), it will describe
it as something mostly reserved for cross-compiling. At first it may seem reasonable, CMake is perfectly capable of detecting
whatever machine and compiler you have and make the decision for you. But the second you start sharing your code with someone
else, or distribute binaries, the whole "on my machine, it works" falls apart. You need to define a common baseline.

In essence, a toolchain configuration is used to describe two things:
* What the code requires to compile (toolchain requirements)
* What machine will be needed to run the produced binaries (target requirements)

The distinction between the two may seem obvious if you've worked on cross-compiled projects, but I've met a lot people who
did not exactly think of them as separate. Perhaps their build infrastructure was always a 100% match with the target production
servers, or simply they never add the "luck" of hitting one of the many issues that can arise from it. Here are some
examples:

* Compiler optimizations may use CPU features that are available on the host but not the target
* Runtime libraries (libc, libstdc++...) may differ between host and target, resulting in potentially incompatible binaries
* Debug binaries cannot be distributed outside the build/test machines on some platforms (Windows MSVC for example)

### A toolchain configuration explained

Let's see an example of such configuration, coming from the default toolchain configuration I use with [Conan](https://www.conan.io/):

```
arch: x86_64
build_type: Release
compiler: Visual Studio
compiler.runtime: MD
compiler.version: 15
os: Windows
```

Here we declare that we're building for 64 bits x86 hardware, in Release mode, with Visual C++ 2017 using the shared runtime.
As you can see the distinction between toolchain and target requirement isn't explicitly stated. This is quite common with build systems.
Let's walk over each one:

* `arch` is a target requirement. It is not mandatory that the host machine fits the bill.
* `build_type` is a bit special because it's more of a directive than a requirement, as I don't know of any system where building
  with or without optimizing and with or without debug symbols affects compatibility.
* `compiler` and `compiler.version` are toolchain requirements, here we need Visual Studio 2017 to compile.
* `compiler.runtime` is a target requirement, we will build binaries that link to MSVC's dynamic runtime. The version of the
  runtime is a hidden target requirement here, derived from `compiler.version`. Technically we build for version 14 (the one associated
  with VC++ 2017) and won't be compatible with older releases.
* `os` is obviously a target requirement, we are building programs for Windows. Again this could possibly be refined with a `version`
  should the operating system have different binary formats depending on the release. And as for `arch`, it is not a strong requirement
  for the host, which could possibly run another OS.

Still with me? I hope it's not too much to think about already, because technically we left some things unsaid here. For example,
we didn't say *which* version of the C++ standard we wanted. Should the build system assume we want the latest available?
Should it let the compiler use its default? Do we know which threading or exception model will be used? Should we care?

The good news is, we (the toolchain config maintainers) don't always have to care. It's enough for a start to define the
minimum requirements that suit our needs and add the rest only if a specific reason arises.

The bad news is, things are not as simple as this, because most build systems out there won't cooperate as we expect.
Namely, two problems are likely to appear: toolchain configuration override and inability to use it as is. Let me explain.

### Read-only configuration?

Here's the thing: most build systems try to be smarter than they should. Given a toolchain which defaults to C++03 and
a project that requires C++14, they are likely to change the `-std` flag to satisfy the constraint. Given a
`x86_64` target architecture and a host with a Haswell CPU, they may try to use AVX2 instructions which are not available
on previous generations. Both might produce binaries incompatible with the target defined in the toolchain configuration.

This is not always entirely the build system's fault. Sometimes it simply *allows* a project maintainer to change the
toolchain configuration, but won't do it if not instructed to.

To me, both are wrong. The whole point of a toolchain configuration is to define a common ground for all the binaries built
for it. Changing it behind the scenes may result in binaries not usable by all targeted clients, or even incompatible binaries within
the same project (for example if two libraries overwrite with incompatible settings). A build system should only allow the project
maintainer to *require* some settings (failing to build if they are not met) or *recommend* them
(raising warnings if they are not met).

This does not have to be the default behaviour, but at least some kind of "strict" mode should be available with a toggle flag.
Without it, users of published libraries will always take a risk when adding a new one to their project: that it would bring
hidden compatibility issues. As I already said in my previous article, it will be a serious challenge for package managers
and any initiative to introduce a common ecosystem in C++ if each library in the build can change those settings.

### Settings vs compiler flags

You may have wondered how Conan derived the right compile flags from the settings I shown in the previous example.
The truth is it doesn't. Not entirely at least. For example, the `arch` will be used to pass `-m32` or `-m64`
to the compiler, but finer settings will not because each one of them will require dedicated code in the package
manager. Instead, you would put a freetext setting to describe your profile and then you would need to add some
environment variables manually under the `[env]` section.

Build systems suffer from the same issue. At best they might know the most common used flags and expose them as settings
(for example CMake can set the C++ standard revision through the `CXX_STANDARD` property). And when you're trying to mix
different libraries with different build systems, all of them need to understand those settings.

This glaring issue with C++ builds was well described by Isabella Muerte at [CppCon 2017](https://youtu.be/7THzO-D0ta4):
most build systems see toolchain settings a bunch of strings called `CXXFLAGS` and `LDFLAGS` which they can't reason about.

Worse, since those flags contain both global toolchain settings (target cpu, runtime, standard...) and local project flags
(include directory, defines, warnings...), build systems can't easily implement the strict mode we talked about in the previous
section because that would require an ability to distinguish between which flags are OK to change and which are not.

### Standardizing Settings

The solution proposed by Isabella is the Universal Compiler Interface (UCI), a program or library that can translate
build settings into flags for all compilers. It could be used by any build system to invoke the compiler with the right flags.

Better: it gives us a common standard to describe the toolchain and build settings of any project. If you remember
my previous article, I described a possible interface between package managers and build systems. The first
step in the process was for the package managers to describe what I called the "build environment" and see
if the build system of a library could work with it or not. That message requires a common language, one that the
UCI language could fit perfectly. 

Obviously, this is a huge task, so I would first concentrate on anything that has an effect on ABI:
* Architecture (CPU family and generation)
* C & C++ Runtime (including version)
* Operating system (including version too if needed)
* Sanitizer / profiler (`-fsanitize`, `-profile`...)
* Threading model, exception model, calling convention, mangling convention...

And since I don't know of any project that describes its toolchain by stating the ABI requirements and letting the build system
pick any suitable compiler, I would also add the most common settings not already listed:
* Compiler (including version)
* C++ standard revision
* Inclusion of debug symbols
* Optimization level

Of course, there would be two sets of toolchain settings, `host` and `target`, to handle cross compilation. If only one is given,
they are assumed to be the same. If both are given, the first one is used to build intermediary tools that may
serve the build process (such as code generators) and the second one for the resultant binaries.

Finally, some settings should be available to build systems (but not toolchain configurations) to invoke the compiler:
* Target type (object file, library or executable)
* Source file(s)
* Preprocessor defines
* Include search directories
* Library search directories
* Libraries to link
* Linkage (static vs shared)

Some of you will probably remark that this looks a lot like `libtool` and you would be right. There are similiarities, except
that here we handle more C++ specificities and try to work with all compilers and platforms, not only the GNU ones. Also,
we only concern ourselves with compiler invocation, while `libtool` also handles install and other things we prefer to leave
to the build system.

### One more step towards a portable C++ ecosystem

In retrospect, it should not be surprising that our analysis of package managers and build systems has come to this.
Before we have a portable C++ ecosystem, we need a portable package manager. And for that we need a standard way
to talk to build systems. And for that, we need a common language to describe the toolchain.

On one hand, it's a bit scary that in 2018 we don't have one already. On the other, we see that solving this issue will
also benefit existing build systems by providing a tool to externalize compiler invocation and focus better on what we
expect from a modern build system (proper handling of dependencies, compilation graphs and incremental recompilation).

And wouldn't it be nice to stop pretending we [do header-only libraries with 0 dependencies because it's trendy](https://youtu.be/XWRbbTVcZwQ)?
