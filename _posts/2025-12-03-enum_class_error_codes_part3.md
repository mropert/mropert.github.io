---
layout: post
title: "C++ Enum Class and Error Codes, part 3"
tags: [cpp]
description: > 
  Last time we explored some commonly found alternatives to using enums for error codes such as asserts, contracts and std::expected.
  Finally today we consider some unthinkable options.
author: mropert
---

The code examples given in [part 1](/2025/11/11/enum_class_error_codes/) and [part 2](/2025/11/17/enum_class_error_codes_part2/)
strongly focused on simple (and I'd argue, quite common) form of error handling: run a sequence of operations, and if any
of them fails bail out and report an error. We tried a few alternatives but they all came with caveats. Error code
returns polluted a lot of the logic with `if ( error ) { return error; }`, `std::expected` was a bit more
concise but demanded we rewrite the signature of the functions we call to fit the monadic form and finally
contracts allowed for cleaner user code but aren't really made to do custom error handling 
([if they make it into C++26 at all](https://wg21.link/p3851)) or recoverable errors (like a missing file).

As it turns out, there is a way in C++ to write a function or block expecting everything will succeed, and then
through some compiler magic have the control flow bail on error and return, while still running any destructor that
needs to be run. In fact it's been there since the start. We could just have the functions we call throw
an exception if they fail.

## Exceptions? Really?!

Hear me out for a second. On paper this seems to fit the bill. Let's look at what it would look like:

```cpp
Model load_model( const std::filesystem::path& path )
{
    const auto blob read_from_path( path );
    const auto asset = parse_gltf( blob );
    return upload_to_gpu( asset );
}
```

For the implementer of `load_model()`, this is about as concise and clean at it could be. We write as if no error could
happen, focusing on the happy path. And for the caller of `load_model()`, they can either add a `try`/`catch` block to
handle failures, or let it pass through to our global exception handler (or call std::abort() if we don't have one).
In this case the caller should probably catch and handle it, as loading assets is the kind of thing that should be allowed to
fail (although it should be loud about it so it gets fixed).

We could even one-line it like this:

```cpp
Model load_model( const std::filesystem::path& path )
{
    return upload_to_gpu( parse_gltf( read_from_path( path ) ) );
}
```

Now for the drawbacks...

## "Exceptions are slow and bloat the code"

This is the kind of thing that keeps being told and retold, but no one ever really checks or tries to reproduce. We just
assume it's true and move on. Luckily, over the past couple years someone took upon themselves to actually verify
those assumptions.

That's the work Khalil Estell has been doing for a few years now, with results shown in 2 talks I recommend looking at:
* [C++ Exceptions are Code Compression](https://www.youtube.com/watch?v=LorcxyJ9zr4)
* [Cutting C++ Exception Time by +90%?](https://www.youtube.com/watch?v=wNPfs8aQ4oo)

It focuses on GCC, ARM and embedded development, three things I don't interact much with (after all PC games mostly care about
x86_64 Windows Desktop), but the learnings and results are still applicable. As it turns out, there's a lot of FUD out there
about C++ exceptions, and a lot of conventional wisdom dates back from the pre 64 bits era, when exception handling was done
through dynamic registration (`setjmp`/`longjmp`) and not using the more modern table driven approach.

As Khalil explains in his talks, the table approach used in x64 (and ARM64) has barely any cost on the golden path,
it mostly manifests as a bit of extra code for `catch` blocks that get correctly branch predicted by the CPU. There
is a cost to handling caught exceptions, but there again there's a lot of room for improvement (as shown in the second
talk, GCC's exception handler had a 20 years old `// TODO: optimize this` comment in its source code).

In our case, having some overhead when handling exceptional failures such as an asset failing to load doesn't seem like
a big deal either way.

## The real pros and cons

The main value to me has already been described earlier, it makes everything concise. The best code is no code, and in this case
we don't need to write _anything_ outside of when we want to handle exceptions. The implicit pass-through nature is what
allows us to focus on what the code is trying to do, and let the compiler handle error case propagation for us. This is the
kind of quality of life thing I enjoy about C++ and other high-level languages.

The other advantage I've noticed is that unlike `std::expected` and enum error codes, we can finally have constructors that can fail.
Gone are the days of two phase construction and `.init()`, or the ugly post construction `.fail()` check to make sure the object
is actually useable. Not every constructor needs to be able to throw but for those few that can fail (like a class that encapsulates
a hardware device) this feels much cleaner.

The biggest issue with exceptions, in my book, is the lack of documentation in code and enforcement by the compiler.
One cannot tell if, and more importantly what, a function may throw just by looking at the signature.
The use of the `throw` keyword in declarations was deprecated in C++11 and removed in C++17. The only thing that remains is `noexcept`,
and quite frankly I find it useless for documentation. In my experience it's only used for optimization purposes to satisfy
`if constexpr` checks like `std::is_nothrow_constructible` and friends.
I know that ship has sailed, but in my book `noexcept` should have been the default and we should have kept `throw` as a mandatory
declaration for the functions that need to throw exceptions.

Since C++ didn't go the Java route (which honestly I think they got right with the `throws` keyword), we're left with comments
to know when a try/catch block might be necessary. This is bad because it forces humans to rely on documentation (or read the
implementation) rather than rely on something that the compiler can check and enforce (the Java compiler will force you to either
catch all exceptions declared thrown by a called function, or declare the calling function as throwing them too).

Why did we decide to go this way in C++? Frankly I'm not sure. The references I could find didn't have very convincing arguments. It
was argued in [P0003](https://wg21.link/p0003) that "exceptions specifications are a failed experiment", but it's unclear to me whether
it's just an observation that the poor original design lead us there, or because the author doesn't like the idea at all.
The original sin seems to date back from 1995 where [N0741](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/1995/N0741.htm)
favoured not adding any exception specification to the standard library because it would "constrain implementation" and
leave `std::` functions free to throw exception not mentioned in the signature.

C++ didn't opt for checked exceptions, instead making the use of the `throw` specifications optional,
so the standard library decided it wouldn't bother either, and in turn most codebases and libraries did the same.
Perhaps there was a lack of experience with exceptions at the time, Java 1.0 was only released in 1996.
Almost 30 years later we have much more data and we can have a look at whether or not Java developers consider this a bad
decision. I'm no expert in Java, but from what I could see the concerns seemed mostly about which exceptions belongs to
which category, not about the whole system itself.

## Too many exceptions?

In Java, exceptions are split between checked and unchecked. Only the former needs to be declared as thrown and handled,
while the latter do not. In the second group you will find things like `NullPointerException` and `ArrayIndexOutOfBoundsException`.
It's been argued that libraries and framework have gotten their classification wrong on the occasion, but I could
find a [good summary](https://stackoverflow.com/a/19061110) on stack overflow:

> Checked Exceptions should be used for predictable, but unpreventable errors that are reasonable to recover from.

Translated to C++, that rule could be to use exceptions for what Java would consider "checked exceptions", and rely
on contracts, asserts or straight calls to `std::abort()` for the rest. And going back to the error codes that
got this whole discussion started, I think we could derive something similar.

Looking at our `load_model()` example, recoverable errors would be things like missing/inaccessible asset file
or an unsupported format. Something we want to tell the user about (if we're making an editor) or replace with a
placeholder and a screaming log message (if we're running a game test build).

On the other hand we have unrecoverable issues like device failures, programming errors, and generally most of the
errors Vulkan can return. For these all we want to do is invoke some form of custom crash reporting before terminating.
It might be _theoretically_ possible to handle a `VK_ERROR_DEVICE_LOST` by shutting down the whole graphics context and recreating
a new one, but I don't see any game trying to do that rather than going back to desktop and let the player restart the game.

## Final thoughts on error handling

Is it too late for exceptions in C++? Between the weak/non-existent compiler enforcement of `throw` vs `catch` and
all the bad myth spread around about the performance of exceptions, it's certainly not a great environment.

First, I wouldn't use exceptions for what Java consider "unchecked exceptions". In my opinion that's one of the big things
to learn from their usage. Contracts and calls to `report_failure_and_abort()` are better for that job.

Second, after going back and forth between alternatives on my current libraries, I found exceptions made my code more concise
and readable than using `std::expected` or old timey error code return, `enum class` or not.

Finally, thinking back at how it's been done in past projects, the most common was a `bool` return that might be checked,
followed by crashes further down when it wasn't. I'd argue that throwing exceptions with no `catch` block would achieve
the same result, but at least the stack trace would point at the original issue in the crash dump.
