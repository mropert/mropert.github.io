---
layout: post
title: Fifty shades of debug
tags: [build,cpp]
description: >
  Every once in a while, a discussion flares up on social media about C++'s performance in debug.
  But what do we mean when we talk about "debug builds"? The answer might not be as straightforward
  as it seems.
author: mropert
customjs:
 - https://platform.twitter.com/widgets.js
---

"They've done it again", he exclaimed.

"The Committee has made a draft of what will become the next iteration
of C++, and this time again they crammed a bunch of new language features but did not make my code
faster in debug!", he continued.

As the outrage reached its peak, he dared to ask what everybody had been secretly wondering all
along: "What are those guys even paid for?!"

Every once in a while, a similar drama flares up on social media. I could go at lengths to explain
that nobody pays people to attend ISO meetings except (sometimes) their own employers,
or that WG21 doesn't write compilers but this is not the article for it.
For that you can read my [previous entry](/2019/01/02/gamedev_intro_to_modern_cpp/).

Instead today I will argue about semantics.

## The semantics of Debug

The recent episode reminded me that this is not the first time "C++ performance
in debug" is brought up, and that again I wasn't understanding the need. I almost never run a build
of [my game](https://www.paradoxplaza.com/europa-universalis-iv/EUEU04GSK-MASTER.html) in what I'd
consider "Debug". The speed trade off is just not worth what it brings up for my day to day work.
I only run unit tests in debug.

But let's take a step back.

In his great talk, [What Do You Mean?](https://www.youtube.com/watch?v=ndnvOElnyUg), Kevlin Henney
made a good observation about the fact that semantics are quite important when you are trying to make
a point, else your discussion isn't gonna go far.

So when we say "Debug build", what do we mean? As a quick experiment, I ran a poll on Twitter:

<blockquote class="twitter-tweet"><p lang="fr" dir="ltr">In C++, Debug means...</p>
&mdash; Mathieu Ropert (@MatRopert)
<a href="https://twitter.com/MatRopert/status/1156299524379467779?ref_src=twsrc%5Etfw">July 30, 2019</a>
</blockquote>

As you can see from the 287 answers, there is some majority vote for debug symbols, but that's not
exactly what I'd call a consensus. Worse, out of all the options I offered, this is by far the one
that has the least impact on runtime performance compared to full "Release" build!

There was also trick in my question: those 4 options are (almost) entirely orthogonal.
You can mix and match them as you like while turning the others off. A better poll would have
allowed for multiple answers but Twitter is quite limited there, so let's not try to draw too many
conclusions from this.

## Four Axis of Debug

As conveniently laid out in my poll, a "classic" C++ build profile will offer (at least) 4 axis to
play with on the debug spectrum:

* `-O0`, `/Od` and the like control optimization. When disabled, the generated code will strongly
  resemble the C++ source code. No inlining will be performed, no instruction reordering, loop
  unrolling, variable elimination, nothing. This will have a huge impact on performance, as we
  usually write our code for humans first, and machines second.
* `-g`, `/Zi` and friends decide whether or not the compiler emits debug information. The process
  of compilation being very lossy, the debugger needs extra data to be able to link back machine
  code to source code when debugging. That database can be stored outside the actual binary, and has
  no impact on performance by itself.
* `/MTd`, `/MDd` will switch the C++ runtime to a special debug version that contains extra checks.
  On Windows, the debug runtime will help catching heap corruptions and bad iterator accesses. It
  comes with a performance cost that is not nearly as bad as it looks *if* optimizations are enabled
  *and* the standard algorithms are used. Contrary to the two previous options it has ABI
  implications and should be turned on for all the objects in the binary (including dynamic libraries).
* `-UNDEBUG` (or the absence of `NDEBUG` in defines) ensures asserts are built in. The `assert()`
  macro function is required to only produce runtime checks if `NDEBUG` isn't defined. The impact
  will greatly depend on the codebase usage of it. A badly placed assert in a tight loop can
  kill performance for good, whereas a sanity check on the inputs of a database API will probably
  not have a noticeable effect.

As previously mentioned, those 4 options can be mixed and matched depending on what's needed and
which tradeoffs are acceptable.

It's important I think to notice that the most sought after feature, debug symbols, have no impact
on performance and can even be stored outside the program for added isolation. For example it is
customary for libraries on Windows to ship `.pdb` files alongside `.dll` files so that developers can
get some modicum of information even in release mode.

I am genuinely curious as to why it came up as the most strongly associated with Debug while it
is in fact probably the least tied to the "performance in debug" debate.
A small suspicion I have is that it could be due to CMake
defaults, in which the `Release` profile doesn't include debug symbols. In which case let me tell
you, dear reader, that CMake defaults are wrong and you should override them. For example, if you
create a project using MSVC's wizard, it will only generate 2 profiles, `Debug` and `Release`, both
of which produce debug symbols.

## Debug runtimes and ABI fun

The second one, the debug runtime (`/MDd` or `/MTd`) is a bit more complex. In my opinion its
biggest impact is on compile time. Since it's potentially an ABI breaker, all of the program
(including 3rd parties) should be recompiled using that flag. In a classic workflow where the
programmer suddenly realizes he needs the debug runtime to track a nasty bug, the thought of needing
to recompile *everything* can be quite disheartening. And then, there's the runtime performance cost
of checked iterators.

The compilation issue can be partially mitigated by a good build system and package manager. Simply
put, there is usually a lot of code in a project that doesn't need recompiling often and could
be cached. Third parties are the obvious example, but low level libraries,
frameworks and other engines could also probably be built every night and shared across
developers. Readers who want to know more about the state of package management in C++
can go and watch [this guy talking about it at ACCU](https://www.youtube.com/watch?v=k99_qbB2FvM).

As for the performance impact, I ran my own benchmarks and discovered that it is indeed utterly
terrible in tight loops... when used with custom algorithms. As it turns out, the optimizer
has a hard time lifting iterator validity checks outside of the inner loop, leading
to extremely redundant checks. This is why MSVC's STL code has iterator debugging aware code in
`<algorithm>` and `<numeric>` to keep performance decent.

A brief glance Visual C++ 2017's implementation of `std::accumulate()` for example will reveal
that an initial check is made on the validity of `first` and `last` parameters, and then
unwrapped iterators are used in the tight `while()` loop to ensure vectorization can still happen.
C++11 range-for loops also seem to be optimized properly, meaning the big thing to keep an eye about
is hand-made specific code that runs tight loops using STL iterators.

## Help me, /Ob1 Kenobi

Disabling optimizations in C++ is, of course, a serious tradeoff. Sure, no inlining or vectorizing
makes the step through code process easier when debugging, but it comes at the price 
of most of the zero cost abstraction principle.

Fortunately, compiler vendors do not consider optimization an "all or nothing" deal and offer ways
for the user to determine which tradeoff they are willing to accept. Clang and GCC, for instance,
have the `-Og` flag to enable all optimizations that don't interfere with debugging. MSVC doesn't
have the equivalent but the compiler has an "optimized debug" feature (`/Zo`) that is on by default
and include extra debug info to [track local variables and inline functions](https://docs.microsoft.com/en-us/cpp/build/reference/zo-enhance-optimized-debugging?view=vs-2019).
As MSVC users might tell, results may vary...

Another option for MSVC is to disable optimization (`/Od`) but keep a degree of inlining so that
the STL stays reasonably fast: `/Ob1`. What it does is allow inlining of functions declared `inline`
or using the `__forceinline` macro. Incidentally the STL is tagged accordingly so that most of it
can still be inlined. CMake's default `RelWithDebInfo` build profile keeps the optimization flag
(`/O2`) but lowers down the inlining from `/Ob2` (inline all the things) to `/Ob1`.

It's important to note that optimization flags do not break ABI, meaning it's perfectly fine
to build 3rd parties and other "stable" code with full optimizations, while keeping the user's
project with less aggressive optimization settings. Again here a package manager would help,
as it should allow the user to use a release toolchain for dependencies while keeping with a debug
toolchain for the local project (as long as they only differ through optimization flags).

Another option would be to make external code inlined, while user's code isn't. A simple way would
be similar to the `dllimport` / `dllexport` preprocessor trick: a macro that expands to either
`__inline ` or `__forceinline` if no particular define is set (used from the outside) or to nothing
when that define is set (when that library is being built for debug). This way the `/Ob1` flag would
make the compiler inline code from external sources only.

A more reliable (and less intrusive) solution would require compiler support for an inlining level in which
code found through `-isystem` inclusion (or module `import`) would be candidate for inlining, but 
code found through `-I` search path wouldn't. One would then just need to tell CMake to translate
`INCLUDE_DIRECTORIES` properties to `-I` whereas `INTERFACE_INCLUDE_DIRECTORIES` would be pulled
though `-isystem` (or equivalent). A package manager could also ensure that dependencies include
directories are passed through `-isystem`. Both solutions should be discussed with compiler and
build systems vendors and not through WG21 as inlining isn't defined in the C++ Standard (well
the keyword is, but only in the context of linkage).

Oh and I'd like to quickly mention `-fomit-frame-pointer`, the dreaded optimization that breaks
debuggers. For one it doesn't, as the C++ standard requires the runtime to be able to unwind
local variables if an exception is thrown, meaning the debugger will also be able to find them.
Secondly, omitting the frame pointer is a dubious optimization anyway as the x86 and x86_64 define
`RBP` as non-volatile, meaning a function that uses it will have to restore it so it only really
saves a `push` and a `pop` in select cases, for total of probably less than 10 x86 cycles (as
the stack is probably in L1 cache anyway). Benchmarking is advised before adding this flag to a
project build profile.

## assert(fibonacci(10000) == 3e10^2089))

The standard `assert()` macro is disabled by setting the `NDEBUG` preprocessor define. The standard
requires it. What it doesn't require is for optimizations to disable it.

Contrary to popular belief, passing `-O2` or `/O2` does not imply `-DNDEBUG` or `/DNDEBUG`. The
two are orthogonal, but build systems usually put one with the other when generating pre-defined
profiles. For example CMake defines `NDEBUG` in both `Release` and `RelWithDebInfo`.

Which means, again, that one can perfectly enable asserts in a build profile other than `Debug`.
I'm not advocating for enabling asserts in release (I don't think anyone does?) but there's no
reason why a tailor-made "development" profile couldn't have a modicum of optimization while keeping
asserts (and debug symbols) on.

When it comes to usage, the only things to really watch out for are dubious asserts that have side effects,
heavy computations or that are run in tight critical loops. In my experience the reason
they may happen is precisely because so many default profiles disable them, leading to them been
rarely seen.

# Conclusion

As we have seen, build profiles are not a boolean that can be either `Debug` or `Release`.
Compiler vendors know that this is a complex problem that is unlikely to have one solution and
offer a myriad a settings to play with.

For the same reason the C++ Standard will probably never specify what `Debug` means or how fast
it should run.

And as my regular readers probably know by now, there are many things we complain about regularly
that end up being a build system issue more than anything. This one is just another in the list.
There is a much to be done by figuring which build profile is good for one's use case, formalizing
it as a toolchain file, and making sure one's build can correctly use it. So go out there and clean
those `CMakeLists.txt`!