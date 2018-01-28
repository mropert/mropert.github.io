---
layout: post
title: PIC, PIE and Sanitizers
tags: [build, cpp]
description: >
  Position-independent code (PIC) and Position-indendendent executables (PIEs) are nothing new, yet they are still a bit obscure
  compile toggles that you ignore until you can't. Luckily for my readers, I had too un-ignore them to make things work.
  Here's the after action report.
author: mropert
---

Last week I was stuck chasing an annoying bug. You know the kind: random crash, happens about once every 20 or 30 runs
when the CI runs unit tests, can't be reproduced by hand, doesn't show-up on [Valgrind](http://valgrind.org/)...

My first idea was to try something more aggressive. I recompiled my code with
[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) (ASAN) which usually catches things
that Valgrind didn't, but still no luck.

Next step was to also recompile externals deps with ASAN, because I suspected that maybe the issue was happening inside them
(possibly due to a faulty use of mine) and ASAN can only find issues that happens inside the code built with it.
Of course in C++ we're still figuring out how to make reusable packages that work with usual flags, so almost none of them
has a simple way to work with ASAN. I'll come back to that part later.

Facing the perspective of needing to patch a couple Conan recipes for ASAN, I took a break and started to wonder if maybe
the issue was a threading issue. Race conditions could explain why it was near impossible to reproduce.
Fortunately ASAN is not the only one of its kind, there's also (among others)
(ThreadSanitizer)[https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual] (TSAN) that specializes in data
races bugs.

I rebuilt my test with `-fsanitize=thread` and went on my merry way:

...

Oops, I forgot to add `-pie` and `-fPIE` as suggested in the doc. Let's try again:

...

Still not good. My gtest library was not built for position-independent code :(

### Pics and pies

Position-independent code (PIC) is a very old thing (in computer science terms, of course). Back before we had MMUs and paging
all code had to share physical addresses, so it quickly made sense to make sure a program could be loaded at any address.
When CPUs evolved and each process could have its own logical address space arbitrarly mapped to any physical address, PIC
was not mandatory anymore.

It came back with shared libraries. The reason we are taught early to put `-fPIC` when we build `.so` files is to ensure that
the same code can be used by multiple process which will map the same physical memory to various virtual addresses.
Praticaly it eliminates all absolute addressing, replacing it with either relative addressing or a small jump table that
can be forked for each process. It also comes with a small cost, which is why it's not always enabled by default (more about that
later).

Before the early 2000s, PIC was limited to shared libaries. Executables still used absolute addressing as you can depend
on PIC when you're not PIC, but not the other way around. With the advent of
[Address Space Layout Randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization) (ASLR), it started
making sense to also make executables Position Independent so that their memory map could not be predicted by attackers,
rendering buffer-overflow exploits much more complicated to pull-out.

Operating systems today will check if an executable is Position Independent (PIE) and if so enable ASLR. Your compiler
may or may not enforce this by default depending on your system.

In practice Position Independent shared libraries are created from objects built with `-fPIC` and
Position Independent Executables are created from objects built with `-fPIE`.

### Mixing-up binaries

Of course since we're doing native code, this means this is yet another thing to keep in mind to determine whether or not
binaries can work together. The [documentation](https://gcc.gnu.org/onlinedocs/gcc-7.2.0/gcc/Code-Gen-Options.html#Code-Gen-Options)
can be hard to process so here's a summary:
* Position-Independent executables may only be created from PIE objects (`.o`) or PIE static libraries (`.a`)
* PIC shared libraries can be created from PIC or PIE objects or static libraries
* Non-PIE executables(*) can be created from any objects or static libraries

In practice, the most simple solution would be to build everything with `-fPIE` and be done with it. But this is C++, we
like to brag about the fact that we only pay for what we use, so obviously a finer rule may be needed:
* If the object is to be linked as a shared library, or a static library linked to a shared library, use `-fPIC`
* If the object is to be linked as a position indenpendent executable, or a static library linked to a
  position independent executable, use `-fPIE`
* Else, use neither

### The obligatory paragraph about build systems and package managers

Ideally, one would prefer to just specify what it wants (a shared library, a static library, a position-independent executable)
and let someone else figure out which flag to use where. A build system for example...

In practice, support is still limited. In CMake for example you can set the `POSITION_INDEPENDENT_CODE` property on a target
and let it set the right flags. It's even on by default for shared libraries. Unfortunately, it only sets the flags
for the compiler, not for the linker (it lacks `-pie` when linking executables), meaning the result binary will not be
a PIE. There's been an issue opened for some time:
[https://gitlab.kitware.com/cmake/cmake/issues/14983](https://gitlab.kitware.com/cmake/cmake/issues/14983).

At package management level, Conan has no support for PIC/PIE either. You could try add the profile to your profile but it
wouldn't work, since PIE requires to add `-pie` to link flags only when linking executables, so if you add to your `LDFLAGS`
you won't be able to link shared libraries anymore. A more native setting would be required, that would set the right
flags for each generator (CMake, Autotools...) and possibly some workaround in package recipes too.

In my case, I "solved the issue" by enforcing `-fPIE` in my Conan profile, then patching the recipe of the final executable
to add `-pie` by hand using `CMAKE_EXE_LINKER_FLAGS`. It wasn't pretty and would probably be hard to scale but it did the job.

### Performance and final words

Position Independent Code is a very useful feature that's been here for a long time, but is still weakly supported by our
C++ ecosystem. Unless enforced as the default by the compiler (you can build gcc/clang that way) it is quite painful
to deploy as soon as you start dealing with projects that mix-up various libraries. The issue is further complicated
by the fact that it works in "reverse": you must know how your code will be finally linked at the time you compile it
(something you probably can't know if you publish a library).

Package managers (and to a lesser extent build systems) can help with that but are still far from perfect on the matter.

In the end the easiest solution might simply be to always compile with `-fPIE`, and let the final linker decide if it wants
a PIE or not. I did not run the benchmarks, but it seems like the impact is negligible for general purpose x86_64 code.
It is not as easy on x86 32 bits: according to [this paper](http://nebelwelt.net/publications/files/12TRpie.pdf),
you should expect a 10% overhead on average.

As for my random bug, as it turned out TSAN didn't find anything either, but at least I got a topic for my blog :)
