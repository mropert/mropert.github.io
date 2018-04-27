---
layout: post
title: Simplifying build in C++ (part 3)
tags: [build, cpp]
description: >
  Package management and build systems are one of the next big challenges C++ is going to face.
  In this article series, I offer some thoughts and ideas to solve that.
  This post is the third part, where we talk about a project's build itself and how simple it could be.
author: mropert
---

A couple months ago I started a series of posts about build systems and package management.
The [first part](/2017/10/19/simplifying_build-part1/) explained the general outline of the issue and a potential solution.
The [second one](/2018/01/06/simplifying_build-part2/) delved a bit more in the details of toolchains and their configuration.
In this third article, we will try to explore the description of a project itself.

## Anything to declare?

There are usually two flavours of a build configuration you can find out there: scipted or declarative. The first one
works like a script or program which execution produces the build artifacts (programs, libraries...) and the second
one is simply a set of values describing the project that the build system can put together to know how to build it.

The big difference between the two is that the first one is much more complex. Since it's a program by itself,
it's very hard to analyze for correctness and can't easily be modified by a machine to handle something the author
didn't expect. The more logic and branches it contains, the higher the chance that it will fail on a particular setup
or environment.

The question then is: why do we do this? Why are the majority of C++ builds defined by some script (CMake being the most
common)? The historical reason is portability. While the build itself is usually pretty declarative (think about good old
`make` rules), the configuration part usually involves lots of checks, branches and switches.

But do we still need that today? In my experience I saw mostly 3 reasons why configuration was needed:
* Lack of standard, or bad implementation thereof. This is why `./configure` still checks for the existence of `<stdio.h>`
  and `printf()` in 2018. For a long time there were a lot of bad/buggy compilers out there, but I believe that time is
  beyond us now. Both GCC, Clang and MSVC have a pretty solid implementation of ISO C++17 and shortcomings are supposed
  to be reported as bugs, not used as features.
* No toolchain definition. As we seen in the previous post, the definition of which build tools (including compiler) are
  to be used is paramount. Configuration scripts would try to compensate by looking for `cc`, `ld` and friends in
  various places. Then they had to figure out the machine architecture and adapt some more flags.
* Absence of package management. Before `conan` and `pkgconfig` and the like, projects that built on other libraries first
  needed to locate them. Then check if the version was suitable. Maybe try to find an alternative. Then adapt some macros
  to tell the sources which cryptographic API to use and whatnot.

With most of those out of the way, I believe there are less and less good reasons to go for a scripted build other
a declarative one. Missing use cases ought to be handled by better standards and tooling instead of resorting to
scripting around the issue.

## A simple build

I've played a bit with my editor and here's an example of what a simple declarative could look like:

```yaml
name: hello
version: 1.0.0
type: lib
standards:
- c++11
- c++14
- c++17
sources:
- src/hello.cpp
public_includes:
- include
public_dependencies:
- boost_asio/1.66.0
tests:
  sources: 
  - test/hello_test.cpp
  dependencies:
  - catch2/2.2.2
```

First of all, yes, this is YAML. If you're allergic to it, you can mentally replace it by JSON or XML or any other document
format that supports maps, arrays and strings.

Let's walk through it:
* First we define the name and version of the project, which would be used to both name the artifacts produced and have a
  unique reference for others to depend on (here `hello/1.0.0`). I will not go into the details of version matching since
  it's a package management concern of its own.
* Next we have the type of artifact being produced: either a library or a binary. Note that I didn't specify the type of
  library. This is usually something you want to leave up to the consumers. The build system should be able to produce
  both static and shared from this same description without more input.
* Following are the standards supported by the project (here C++11, 14 and 17). The build system should check if the provided
  toolchain is compatible with those or not. Why not a minimum? Simply because C++ isn't backward compatible.
  For example a C++11 or 14 project can legally use `auto_ptr` and `random_shuffle()`, whereas a C++17 project can't.
* Then we have the `sources`, the list of files to compile to object files and ultimately link together. Modern projects
  seem to avoid them these days, but I do believe that it is mostly because we don't have a proper build system to
  handle that correctly.
* Include paths come next. In this particular case we only have public (available to consumers) includes but we could also
  have private ones. Individual files could probably be listed like sources and double checked against the declared include
  directories, but I wanted to keep things to a minimum here.
* Our project uses Boost.Asio in it's public headers, so we declare declare it a public dependency. Else it would have
  been `dependencies` or maybe `private_dependencies`.
* Finally we have the tests, with their own `sources` and `dependencies` sections. Here we use Catch2 to test.

With this, I believe we can describe most projects in a simple file, both for build and packaging it to others.
Still, you may find some stuff missing. Let's think about it:

* Files to install. While most could be derived from this definition (libraries, public headers), some additional
  material may be needed, such as documentation and helper scripts. Maybe an `install` section to list those could work.
* Code generation. Things like protobuff or lex/yacc come to mind. This is not something I use everyday but I could
  see some `generated_sources` and `generated_includes` sections added, the trick being to avoid putting command lines
  in. An idea to explore would be to have `generator` packages that can be required and used to transform files
  from format `.xxx` to `.hpp` and `.cpp`.
* Options. I deliberately didn't put any option or conditional here. The rationale being that they make package
  management a nightmare. What if the dependency tree of a project contains the same package twice but with different
  options? Are they compatible? Can one be used in place of the other but not the other way around? vcpkg for example
  decided against options in its packages. This can usually be remedied by splitting the project in multiple ones
  of different abstraction levels. You could for example have a core project, used by both a GUI and a console based
  project that don't need each other.
* Conditional requires. I would like to avoid them too. While I see the value of being able to work with several
  implementations of a dependency (GnuTLS vs OpenSSL for example), I believe dependency injection is a better answer
  here for once. Still, we like to pay for what we use so if you know at compile time which database/security/whatever
  provider you will use, it would be important to be able to use it directly. I'll have to think about this.
* Preprocessor defines. I don't see why they would be needed. The main reason I've used them in the past are options
  (which we are trying to avoid here) and Windows DLL import/export toggles that could be
  [easily generated](https://cmake.org/cmake/help/latest/module/GenerateExportHeader.html) for you by the build
  system depending on context.

Should you have another use case, or disagree with the practicality of the alternatives I suggest, please do contact
me and share it.

## Wait a minute...

Some of you are probably asking yourself "is he trying to write a new build system that also double as a package manager
while [saying the exact opposite at conferences](https://www.youtube.com/watch?v=XWRbbTVcZwQ)?!" and you would be right
to ask.

Fortunately, the answer is no, I'm not writing a new build system on my spare time :)

Instead I'm trying to define a simpler and cleaner format for describing a project to see where the limitations lie
and assess where we should be headed next. I still believe Conan is the most practical solution for a package
manager today and CMake is still my goto choice to build what I'm packaging with it. But I also think that
both are leaning too far on the script side, making progress slow and error prone, discouraging people from getting
into it unless they are build experts. I still see way too many people selling the fact that their library is header-only
as a good thing and that's not a good sign.

My hope is that with a simpler format, we could describe our projects easily, and then  either generate something that
existing package managers and build systems could understand, or patch them to support it natively.

With that goal in mind, I'm now asking for you feedback, dear readers. Tell me if there are limitations you could foresee.
Is there something you do in your current projects that couldn't be expressed with this (that is not due to a design
decision that would probably be challenged by today's standards)? As always, I'm available by email, twitter, and soon
at SwedenCpp too!