---
layout: post
title: "C++ Enum Class and Error Codes"
tags: [cpp]
description: > 
  C++11 gave us `enum class` and while it's great to have scoped enums I don't find it great for error handling. Let's talk about why.
author: mropert
---

Most of my readers, I hope, have been able to use C++11 for a while now (if not hello, I'm sorry the world has changed to become this
weird during your 14 years sleep). With it came a small change that allowed for better scoping of names without resorting to weird quirks:
`enum class`. The idea is simple: owing to C, `enum` values in C++ belong to the class or namespace they are declared in but if we add the
`class` keyword to the declaration they know become their own scope instead of leaking to their parent.

This was a simple quality of life change in the compiler to address a weakness in the language that folks usually worked around by either
adding long prefix to the enum values (C style) or wrapping them within structs or classes (C++98/03 style). And with it came the incentive
to migrate C era error code enums to scoped enums. But there's a catch.

## Testing for error

In my experience the main thing we do with error codes is testing whether or not something failed or succeeded. From C we inherited
the basic test for `false` or zero on a return value:

```cpp
const auto ret = some_operation( ... );
if ( ret )
{
  // Handle failure
}
```

Whether you use OpenSSL, FFMPEG, curl or Vulkan you get the same pattern, the zero value of the error code enum means success, everything
else means an error occurred. Now when we move to C++ realm and use a C++ library or wrapper (like I did with VulkanHpp lately), we get into
trouble:

```cpp
const auto ret = cpp::some_operation( ... );

if ( ret ) // Does not compile, enum class cannot be converted to bool

if ( ret == cpp::result::Success )  // Works, but more cumbersome, I don't like to type
```

Here's the pet peeve that got me writing this article in the first place: `enum class` cannot be evaluated in boolean context. Not even
with an `explicit` constraint.

```cpp
enum class Result {
    Success = 0,
    SomeError,
    SomeOtherError
};

inline explicit operator bool(Result r) { return r != Result::Success; } // Doesn't compile, cannot define operator bool as a static function
```

So, what can be done about this?

## The double bang hack

If you ask stack overflow (or worse, an LLM), you will likely be suggested the double bang pattern:

```cpp
inline bool operator!(Result r) { return r == Result::Success; }

void foo()
{
  const Result ret = cpp::some_operation( ... );
  if ( !!ret ) 
  {
    // Handle errors
  }
}
```

This works and is probably the simplest/least verbose (in terms of code to write to support it) of the various workarounds.
But in my opinion it still looks odd and confusing. If I ran into this in the wild I'd have to stop and wonder if this was intentional or not.
In a code review I'd be likely to ask "Did you mean `if ( !ret )`?". If I wrote it I'd be tempted to add a comment to clarify the intent.
This is the issue with non idiomatic workarounds: they are hard to distinguish from typos and other fat finger mistakes.

## Going back to C++03

Back in the days before `enum class`, there was still a way to scope enums and we could try to leverage that:

```cpp
struct Result {
    enum Value {
        Success = 0,
        SomeError,
        SomeOtherError
    } v;

    explicit operator bool() const { return v != Result::Success; }
};

void foo()
{
  const Result ret = cpp::some_operation( /* ... */ );
  if ( ret ) 
  {
    // Handle errors
  }
}
```

This way we get back our boolean evaluation, but at the cost of loosening the scoped nature of our enum:

```cpp
void bar()
{
    const Result ret = cpp::some_operation( /* ... */ );
    if ( ret.v == Result::SomeError ) {}
    if ( ret.v == 42 ) {} // Oops, does compile but probably shouldn't
}
```

While it's not directly convertible to `int`, the unwrapped code is and can be compared or assigned to any integer which isn't great.
It is still fixable, but at the cost of even more verbosity:

```cpp
struct Result {
    enum class Value {
        Success = 0,
        SomeError,
        SomeOtherError
    } v;

    explicit operator bool() const { return v != Result::Value::Success; }
};

void bar()
{
    const Result ret = cpp::some_operation( /* ... */ );
    if ( ret.v == Result::Value::SomeError ) {} // Works
    if ( ret.v == 42 ) {} // Compile error
}
```

We could shorten `Value` to `V` to keep it concise, but it would still look a bit off, and we still need to use the `.v`
unwrapper unless we decide to go even deeper:

```cpp
struct Result {
    enum class Value {
        Success = 0,
        SomeError,
        SomeOtherError
    } v;

    constexpr Result( Value x ) : v( x ) {}
    constexpr explicit operator bool() const { return v != Result::Success; }

    static constexpr Value Success = Value::Success;
    static constexpr Value SomeError = Value::SomeError;
    static constexpr Value SomeOtherError = Value::SomeOtherError;
    // Repeat for each value in the enum
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

Now we finally have our feature set back with no extra verbosity on the client side, at the cost of a lot code on the API
side, plus some absolutely horrendous codegen when optimizations are off. Even with the `constexpr` keywords everywhere.

## Put it in the compiler

This could be fixed if the standard allow to define an explicit `operator bool` for `enum class`es. But I believe the issue goes wider.

This situation illustrates a recurring beef I have with C++ design: delegating strong typing to library code with questionable first class language support.
I have the exact same issue with units libraries and pointer/indices wrappers. It should "behave like `int`" but it doesn't unless
I add `/O2 /Ob2` or `-O3` or something.

If at least the standard guaranteed that wrapping machine word sized native types within a standard layout class or struct will translate to the _same_ code
as if I used the unwrapped type (be it `int`, `float` or pointer) _even without turning optimizations on_ then maybe we could talk.
Or if at least every major compiler had a special optimization flag that I could turn on to inline all those then we'd be cool (or at least OK).

I am aware that this is technically a "quality of implementation" issue and a digression from the original topic but I felt like I had to mention
this after looking at the assembly generated by the previous examples on godbolt. Wrapping error codes in strong types so that we can't mix apples
and oranges is a good thing, but it shouldn't turn a simple "are those 2 integers equal" expression into 2 function calls to keep the client code
on a syntax similar to its C counterpart.

There was more I wanted to address here (can we use something other than enum classes for errors? do we even need that explicit operator bool shortcut
in the first place?) but this post is running long so I will continue on the topic next week. And after that we will resume our "What Makes a Game Tick?"
series. Until then!
