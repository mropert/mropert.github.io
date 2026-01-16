---
layout: post
title: "Designated Initializers, the best feature of C++20"
tags: [cpp]
description: > 
  Of all the features added to C++ over the past years, I think Designated Initializers is both
  the best and one of the least talked about. Time to right that wrong.
author: mropert
---

If you've been following my hot takes on C++, you might have noticed that I haven't been the most enthusiastic
person about the recent additions to the language. While some of them were nice addition, I haven't felt like
they had a significant impact on my code unless I had a somewhat niche use case. But for the past months
I've been using C++20's designated initializers and it's been quite the change.

## The feature

Originally proposed as [P0329](https://wg21.link/P0329), the feature is a port of C99 with some tweaks. 
For those who have never used it in either language, it allows initializing structure members by name
while omitting the ones that should keep their default values.

Here's an example use from my current project:

```cpp
Texture::Desc desc { .format = Texture::Format::R16G16B16A16_SFLOAT,
                     .usage = Texture::Usage::COLOR_ATTACHMENT | Texture::Usage::TRANSFER_SRC,
                     .extent = device.get_extent(),
                     .samples = 4 };
```

And here's the full structure declaration:

```cpp
class Texture
{
    // ...
    struct Desc
    {
      Format format = Format::UNDEFINED;
      Usage usage = Usage::NONE;
      Extent2D extent;
      int mips = 1;
      int samples = 1;
    };
};
```

Note that I am not specifying any value for `mips`. The compiler will leave it to its default value
upon construction, the same way it does for member initializer lists in constructors. There could be 0 or
a dozen members between 2 explicitly initialized elements like `extent` and `samples` and it will compile and work
just fine. But unlike constructor initialization lists, the compiler will emit a hard error if any
of those members appear out of order. This departure in design also makes it differ from the C99 version
which allows members to appear in any order. I think this is a good design choice I know it has its detractors,
more on that later.

## That's all?

This feature might not feel like a big deal at first, especially compared to the other large additions to C++
like modules, coroutines, concepts and the like. So why is it so important in my opinion?

A lot of C++ is about catching bugs with the compiler rather than with the debugger (or the QA process, or worse).
And one of the big ways to do that is with strong types. `Apples` and `Oranges` are two different types that
might both just be `int` under the hood, but if you try to assign one to the other by mistake, you get a compile error.

Going back to my example, before C++17 you could have written it this way:

```cpp
Texture::Desc desc { Texture::Format::R16G16B16A16_SFLOAT,
                     Texture::Usage::COLOR_ATTACHMENT | Texture::Usage::TRANSFER_SRC,
                     device.get_extent(),
                     1,
                     4 };
```

This is good old aggregate initialization, and it's been there forever. But notice how last 2 parameters
are just `int`. Would you immediately catch that the first is `mips` and the second is `samples` without
double checking the struct declaration?

It goes even deeper. Both `Texture::Format` and `Texture::Usage` are `enum class` which in turn, you guessed it,
are also `int`. And why did we make `enum class` in C++11 in the first place? Same reason: to make sure we can't
accidentally mix them up. But you know how else we could avoid mixing them up? Making sure they are used in an
expression with a left hand side _that has a name_.

Compare with old fashioned member-wise assignment:

```cpp
Texture::Desc desc;
desc.format = Texture::Usage::COLOR_ATTACHMENT | Texture::Usage::TRANSFER_SRC;
desc.usage = Texture::Format::R16G16B16A16_SFLOAT;
desc.extent = device.get_extent();
desc.mips = 4;
desc.samples = 1;
```

It's very obvious we mixed things up here, right? Even if `format` and `usage` weren't strong enums, it would
be fairly easy to catch during in a code review. A compile error is nicer, the IDE adding squiggly red lines under
the expression as we type it is even better, but still it's quite jarring.

## C++ Units

But why don't we go the all the way for `mips` and `samples` like we did for `format` and `usage`? That would
be nice if we could express them as different types entirely, no? There are some libraries out there that
offer some options.

```cpp
// With https://github.com/rollbear/strong_type
using Mips = strong::type<uint32_t, struct Mips_>;
using Samples = strong::type<uint32_t, struct Samples_>;

// With https://github.com/joboccara/NamedType
using Mips = NamedType<uint32_t, struct MipsTag>;
using Samples = NamedType<uint32_t, struct SamplesTag>;

// With https://github.com/mpusz/mp-units
// TODO. I gave up after reading too many manual pages
```

Since C++ has no compiler support for named types, they all use similar meta-programming tricks to create a unique
`struct` with some tags that wraps an integer (or similar scalar type). Which usually means a nontrivial amount of library
code dedicated to various operators to bring back all the semantics of `int` to our named `struct`. Compilers
then have various degrees of success translating that back into assembly (mostly fine with optimizations on, mostly
terrible without).

But that doesn't entirely solve our problem, because usually to avoid mixing and matching need to disable implicit
conversion and assignment from `int`, else we miss the entire point of guarding against initializer list mismatch. To fix that
we usually add user defined literal suffixes:

```cpp
Mips operator ""_mips(uint32_t);
Samples operator ""_samples(uint32_t);

const auto format = Texture::Format::R16G16B16A16_SFLOAT;
const auto usage = Texture::Usage::COLOR_ATTACHMENT | Texture::Usage::TRANSFER_SRC;

// Works
Texture::Desc desc1( format, usage, device.get_extent(), 1_mips, 4_samples );

// Compile error
Texture::Desc desc2( format, usage, device.get_extent(), 4_samples, 1_mips );
```

Finally, we've solved it. But don't you notice something? Doesn't `4_samples` and `1_mips` look awfully
close to `.samples = 4` and `.mips = 1`? Except one of them requires an entire strong type library
and the other is _supported natively by the compiler_.

## It goes deeper

So far in our example we've kept to cases where we specified most if not all the members. Or at the very least
we specified the ones that came first in declaration order. But that's not how every struct is laid out.

Let's look at another example from my library:

```cpp
Pipeline::Desc desc { .color_format = draw_image.get_format(),
                      .depth_format = depth_image.get_format(),
                      .push_constants_size = sizeof( push_constants ) };
```

And here's the struct definition as of this article's writing:

```cpp
class Pipeline
{
    // ...
    struct Desc
    {
      // Graphics pipelines only
      Texture::Format color_format;
      Texture::Format depth_format;
      PrimitiveTopology topology = PrimitiveTopology::TRIANGLE_LIST;
      CullMode cull_mode = CullMode::FRONT;
      FrontFace front_face = FrontFace::CLOCKWISE;
      // Compute & graphics pipelines
      uint32_t push_constants_size = 0;
    };
};
```

As you may notice, we are skipping a bunch of values here and leaving them to their defaults. Without
that we would have needed to repeat a lot of code only to say "keep those values as they would be otherwise".

But most importantly, this does not only apply to declaring variables on the stack. Now, we can finally do this:

```cpp
auto depth_tex = device.create_texture( Texture::Desc { .format = Texture::Format::D32_SFLOAT,
                                                        .usage = Texture::Usage::DEPTH_STENCIL_ATTACHMENT,
                                                        .extent = draw_image_extent,
                                                        .samples = 4 } );
```

Or even this:

```cpp
// The compiler will deduce what type we are initializing from the function declaration
auto depth_tex = device.create_texture( { .format = Texture::Format::D32_SFLOAT,
                                          .usage = Texture::Usage::DEPTH_STENCIL_ATTACHMENT,
                                          .extent = draw_image_extent,
                                          .samples = 4 } );
```

Python programmers will look at this and exclaim "Look at what they need to mimic a fraction of our power!".
Indeed, Python has had support for named function arguments since the initial 1.0 release in 1994. Meanwhile
in C++ the best we have is a combinatorial explosion of overloads with default values that cannot
possibly scale.

If we look back at the history of my library the API used to look like this:

```cpp
class Device
{
    raii::Texture create_texture( Texture::Format format,
                                  Texture::Usage usage,
                                  Extent2D extent,
                                  int samples = 1,
                                  int mips = 1 );
};
```

This API would not scale well as it grows and we add more and more options to texture creation, and
adding overloads wouldn't really help.

Is passing a struct and using designed initializers to fill it kind of cheating? Maybe. Is it better
than hoping that C++ will one day have named function parameters? Absolutely. Especially because
it does the job as a byproduct of a feature that is itself already quite neat to use even for initializing
local variables (or any variable really).

## Limitations

The main complaint I have read about this feature is that unlike C99, it doesn't allow for arbitrary ordering.
Worse, if out of order initializers are used in a C header it will fail to compile when included in C++,
regardless of whether it's wrapped in `extern "C"` or not.

On one hand I can see why one would like the rule to be relaxed for types declared as `extern "C"` but sadly this isn't how
it works. `extern "C"` does not revert the language grammar and semantics to C, it only changes linkage. You
can still use any manner of C++ features in an `extern "C"` block and the compiler will be just fine with it
(you probably shouldn't because it will definitely fail to compile with C clients, but technically you can).

The thing is, in C++ I want the order of declaration enforced, the same way I'd like the `clang-tidy`
warning for out of constructor initializer list to be a hard error. In C++ order of construction matters because
constructing an struct member can invoke all manners of side effects and it's not a good idea to mislead the programmer
who might think members will be constructed in the order they are initialized as opposed to the order they are declared.
Enforcing both to be the same sidesteps that issue entirely.

I have pondered the idea of relaxing the ordering rule for [trivial types](https://en.cppreference.com/w/cpp/named_req/TrivialType.html)
but doing so would likely set a type definition in stone, as adding _any_ nontrivial member (or changing its definition to
be nontrivial) would break every client, which is usually not a thing you want in an API.

Another departure from C that's worth mentioning is that since C++ allows for member initializers in declarations,
we can have default values for omitted members that are something other than 0. I found that 1 is also a fairly
common default value (like in texture mips and samples for example) and those cannot be expressed in the C99 version.

To me the only pet-peeve is that designed initializers cannot be forwarded. This does not compile:

```cpp
std::vector<Texture::Desc> v;
v.emplace_back( .format = Texture::Format::D32_SFLOAT,
                .usage = Texture::Usage::DEPTH_STENCIL_ATTACHMENT,
                .extent = draw_image_extent,
                .samples = 4 );
```

You can fix this by wrapping the arguments:

```cpp
v.emplace_back( Texture::Desc{ .format = Texture::Format::D32_SFLOAT,
                               .usage = Texture::Usage::DEPTH_STENCIL_ATTACHMENT,
                               .extent = draw_image_extent,
                               .samples = 4 } );
// Or simply
v.push_back( { .format = Texture::Format::D32_SFLOAT,
               .usage = Texture::Usage::DEPTH_STENCIL_ATTACHMENT,
               .extent = draw_image_extent,
               .samples = 4 } );
```

I've looked at the assembly generated and it's basically the same without or without optimizations, at
least for trivial types, so it's only a minor annoyance, but still worth mentioning.

## Wrapping up

Of all the language features of late, this is the one that I think has (and will) change how I design
APIs the most (remember I said _language_ features, because on the library side there has been things
like `std::span` which has had an impact similar to `std::string_view`).

Only time will tell how this measures up compared to bigger features, but it's a reminder that language
evolution does not have to come with a very big paper to be impactful.
