---
layout: post
title: "C++ Enum Class and Error Codes, part 2"
tags: [cpp]
description: > 
  In our previous article we talked about the limits of using `enum class` for API error reporting. Today we look at alternatives.
author: mropert
---

Last week, we showcased [how C++11's `enum class` can be used for error codes](/2025/11/11/enum_class_error_codes/) and discussed what is,
in my opinion, its biggest limitation: the inability to easily use them in boolean contexts to see if an error did occur. While it
was possible to add the functionality, it required some amount of code to write and didn't generate amazing assembly compared to native
types unless we turned aggressive optimizations on.

I wanted to discuss possible alternatives, but first I want to update on a few reactions I got.

## Keeping it DRY

The great Vittorio Romeo suggested [two](https://x.com/supahvee1234/status/1988266045858046442) [improvements](https://x.com/supahvee1234/status/1988271097226183086).
First, since C++20 we can use `using enum` to avoid the worst of code repetition:

```cpp
struct Result {
    enum class Value {
        Success = 0,
        SomeError,
        SomeOtherError
    } v;

    constexpr Result( Value x ) : v( x ) {}
    constexpr explicit operator bool() const { return v != Result::Success; }

    using enum Value; // pulls all enum names into the scope 
};

inline constexpr bool operator==( Result lhs, Result rhs ) { return lhs.v == rhs.v; }
inline constexpr bool operator!=( Result lhs, Result rhs ) { return lhs.v != rhs.v; }

void foo()
{
    const Result ret = cpp::some_operation( /* ... */ );
    if ( ret ) {} // Works
    if ( ret == Result::SomeError ) {} // Also works
    if ( ret.v == 42 ) {} // Compile error
}
```

If after seeing this you thought "oh that's great, that means I can turn `Result` into a template and only write it once for all enums in my codebase, sadly no.
As it turns out, from [C++20's wording](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1099r5.html): "`using-enum` must name a non-dependent enumeration type".
Too bad.

## Inlining

Vittorio also suggested to add some `[[gnu::always_inline]]` to coax the compiler into doing inlining. I tinkered a bit with it but sadly it quickly
ran into problems when used outside of gcc/clang toolchains:

```cpp
#define FORCEINLINE [[gnu::always_inline]][[msvc::forceinline]]

struct Result {
    enum class Value {
        Success = 0,
        SomeError,
        SomeOtherError
    } v;

    FORCEINLINE constexpr Result( Value x ) : v( x ) {}
    FORCEINLINE constexpr explicit operator bool() const { return v != Result::Success; }

    using enum Value; // pulls all enum names into the scope 
};

FORCEINLINE inline constexpr bool operator==( Result lhs, Result rhs ) { return lhs.v == rhs.v; }
FORCEINLINE inline constexpr bool operator!=( Result lhs, Result rhs ) { return lhs.v != rhs.v; }
```

It works on Clang and GCC but it doesn't behave as you'd expect on MSVC. Here are the problems:
* On GCC and Clang `[[gnu::always_inline]]` is honored even with `-O0`. It's entirely independent of the optimization settings.
* On MSVC, `[[msvc::forceinline]]` is only of use if the inliner is turned on by `/Ob1` or higher, in which case it bypasses the inline evaluation heuristic.
  The thing is, `inline` functions and class method defined inline _already_ satisfy the heuristic and are
  [guaranteed to be inlined with `/Ob1`](https://learn.microsoft.com/en-us/cpp/build/reference/ob-inline-function-expansion?view=msvc-170) so the attribute is useless here.

So already we have a portability issue, but we are not done here:
* Unknown attributes generate warnings in GCC (`Wunknown-attributes`) and MSVC (`C5030`) _even when they use a prefix meant for another compiler_.
  Meaning unless one is willing to suppress all warnings about unknown attributes (including typos likes `[[nodicsard]]` or `[[falltrhough]]`),
  `FORCELINE` must done through the old school `#ifdef _MSC_VER ...` hell as combining all attributes in one would generate warnings.
* Maybe the worst of it all, we now have to clutter our declarations with `FORCELINE` which are already littered with a ton of keywords like
  `constexpr`, `inline`, `explicit` (and probably some `[[nodiscard]]` too if we want to be correct). At some point I start wondering if I should
  have a `#define` collection of shortcut mnemonics like `CFIEND` (Constexpr Force Inline Explicit NoDiscard) to keep declarations readable.

## What if we didn't?

Jason Turner [challenged the entire idea](https://x.com/lefticus/status/1987748567763890426) of needing to convert error codes to a boolean.
He argues that an error code is more than "failure or not" and so shouldn't easily be reduced to that.

I think it is true in _some_ cases, but not all in all. As with exceptions, I think it boils down to whether
or not those errors are recoverable. For example say we're trying to open a file and it fails. Does it matter why?
I would say it matters to the end user and it would be good to tell them why it failed (doesn't exist, busy, no permission...) but
to the code the difference does not matter much. It most likely requires human intervention to be fixed, the program itself
can't do much about it and is mostly concerned with reporting the issue (either by an error pop-up or through some sort of log file).
At best a text editor might be able to offer to flip the read-only bit and then retry.

In the case that prompted this discussion I was working with Vulkan and most errors amounted to "you did something wrong", reporting a logic error.
The only thing to do here is exit or abort and let the programmer fix the bug. The error code is useful as a hint to debug the issue,
not to the program itself. The code within the `if ( error )` branch usually amounts to forwarding the error to some error reporting system
that will either assert or abort.

Which is a good way to bring up potential alternatives

## Contracts

In the case of Vulkan those errors were mostly (late) reports on contract violations. Often a previous API call invoked some undefined behaviour
that put the GPU in a fault state, and a later call returned `VK_ERROR_DEVICE_LOST`. The API was kind enough to return an error code rather than
straight up crashing in the driver, but that's mostly a curtesy (or maybe a smart way of making sure the complaints go to the game developers
and not the driver maintainers when players look at the stack trace).

If you enable the validation layer the API will check for a wide variety of misuses and invoke a callback when they are detected. The common behaviour
I've seen is to log them on the console and continue, but you could technically fire an assert violation, throw an exception or call `std::abort`.
Which incidentally is the purpose of C++26 contracts: detect API pre or post condition violations and handle them through a user-defined action
(assert, abort, ignore or callback).

One may argue that contracts/asserts and error codes aren't the same thing. After all, contracts are there to report bad/illegal usage while
error returns are there for runtime errors raised by proper usage of the API. While this is correct, my whole rant started
about handling errors in Vulkan, and so far most if not all of what I've had to deal with were variations of `EINVAL`.
I have also experienced APIs which basically used error codes to report what should probably have been contract violations.

In that case I think error returns are there mostly to try to gracefully exit with some error logs in release builds
because contracts were disabled for performance reasons, and so there isn't much we care about outside of "did if fail or not?".

## `std::expected`

Since C++23 we have `std::expected<R, E>` in the standard, which is a variant wrapping either an expected return _or_ an error type describing
why the operation failed. This cleans up API design because we no longer need to decide whether a function should return the result object or
an error code and pass the other as output parameter which is what a bunch of API do, especially C ones.

The other big addition of `std::expected` is the ability to chain calls in a more functional/monadic fashion:

```cpp
void foo()
{
    cpp::some_operation( /* ... */  )
        .and_then( cpp:some_other_operation )
        .or_else( error_handler );
}
```

This is quite nice because it allows up to write all the _expected_ code path as one chain of calls, and then define a short circuit for handling error.
In our example of doing graphics, this kind of syntax would allow us to write our entire sequence of operations without interleaving
each call with some "if error log and return".

Compare:

```cpp
io_error read_from_path( const std::filesystem::path&, Blob& );
gltf_error parse_gltf( const Blob&, GltfAsset& );
VkResult upload_to_gpu( const GltfAsset&, Model& );

Model load_model( const std::filesystem::path& path )
{
    Blob blob;
    if ( const auto ret = read_from_path( path, blob ) )
    {
        log( /* Blablabla I/O error */ );
        return {};
    }

    GltfAsset asset;
    if ( const auto ret = parse_gltf( blob, asset ) )
    {
        log( /* Blablabla format error */ );
        return {};
    }

    Model m;
    if ( const auto ret = upload_to_gpu( asset, model ) )
    {
        log( /* Blablabla vulkan error */ );
        return {};
    }

    return m;
}
```

With this:

```cpp
using Error = std::variant<io_error, gltf_error, VkResult>;

std::expected<Blob, Error> read_from_path( const std::filesystem::path& );
std::expected<GltfAsset, Error> parse_gltf( const Blob& );
std::expected<Model, Error> upload_to_gpu( const GltfAsset& );

Model expected_load_model( const std::filesystem::path& path )
{
    Model m;
    return read_from_path( path )
        .and_then( parse_gltf )
        .and_then( upload_to_gpu )
        .or_else( []( const Error& e ) -> std::expected<Model, Error> {
            std::visit( overload{  
                []( io_error e ) { log( /* Blablabla I/O error */ ); },
                []( gltf_error e ) { log( /* Blablabla I/O error */ ); },
                []( VkResult e ) { log( /* Blablabla I/O error */ ); } }, e
            );
            return {};
         } ).value();
}
```

I'd argue the second one looks nicer and might even allows us to write one error handler we can reuse in many places.
It will be slightly slower to compile as the monadic operations on `expected` do a bunch of template instantiations,
although not nearly as bad as `<ranges>`. The generated code will also be quite worse without optimizations
(and still not great even with optimizations on MSVC), but this might be acceptable since the cost of orchestrating those operations
remains negligible compared to the work each operation does.

For more details on functional approaches to error handling and the like, I recommend [Jonathan Müller's talk](https://www.youtube.com/watch?v=lvlXgSK03D4).

## Anything else?

There is one last options I have been toying with lately but haven't mentioned here (there's a hint _somewhere_ in this post)
but given the controversial nature of the topic and the current line count, this will have to wait for next week.
See you there!
