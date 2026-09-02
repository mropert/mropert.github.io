---
layout: post
title: "Make APIs that fail gracefully rather that fallback silently"
tags: [cpp,gamedev, graphics]
description: > 
  A tale of many hours spent trying to find a bug that was hidden by an
  API with a silent fallback.
author: mropert
---

I have been on and off [toying with my own 3d renderer](/2025/12/30/a_year_with_graphics/) since
last year. I started on Windows but recently I have tried to validate the cross-platform
capabilities by porting it to Mac. I had to fight off the limitation of MoltenVK for a while,
but the real issue came from a different source.

For weeks, if not more, my lighting/colours were off. And when I thought I had finally fixed
them to get something I was happy with on Mac, it turned out completely off and washed out on
Windows. Back and forth and back and forth.

The bug turned out to be connected to what I would qualify as bad API design. But before
we get there, I need to make a small digression.

## The AI subplot

Coming off my [recent article](/2026/08/04/an_honest_review_of_ai_programming/), I felt like
maybe Claude could help me find the issue. After all, [colours are hard](https://www.youtube.com/watch?v=_zQ_uBAHA4A).
But in asking AI to solve this I made the mistake of using it to solve something I wasn't enough of an expert in.
So it took me on a wild ride.

Here's what it suggested I fix, rewrite or implement to address the perceived issue:
* Rewrite the PBR light accumulation shader and all the math that came with it
* Replace my fake ambient factor by IBL or at least a placeholder cubemap
* Use a fullscreen shader to copy from my HDR render texture to the SDR swapchain instead of doing a blit
* Move UI rendering to a different pass and render target
* Add a tone mapping/gamma correction pass
* Replace the added Reinhard tone mapping with a filmic ACES filter with an exposure dial

Would you care to guess which one of these was the source of my bug? None of them.

The bug was that [vk-bootstrap](https://github.com/charles-lunarg/vk-bootstrap) defaults
to creating a `VK_FORMAT_B8G8R8A8_UNORM` swapchain<sup>[1](#myfootnote1)</sup> on Mac if the format you request
is unavailable.
So color grading on Mac would make everything too bright and washed out on Windows since
it applied hardware gamma correction but Mac didn't, and vice versa.

It's a personal project and learning is the main objective so I didn't mind the rabbit hole
too much, but on a more serious deadline I'd hate to have my timeline derailed for days
by AI going entirely the wrong direction. And then be presented with the token bill.

And with that parenthesis out of the way, let's get back to the main point of this article.

## Error: operation failed successfully

Here's roughly the code I used to setup my swapchain:

```cpp
vkb::SwapchainBuilder builder( *device._physical_device,
                                *device._device,
                                *device._surface,
                                device._gfx_queue_family_index,
                                device._present_queue_family_index );
builder.set_desired_format(
    VkSurfaceFormatKHR { .format = VK_FORMAT_R8G8B8A8_SRGB,
                         .colorSpace = VK_COLOR_SPACE_SRGB_NONLINEAR_KHR } );
builder.set_desired_present_mode( VK_PRESENT_MODE_FIFO_KHR );
builder.set_desired_extent( device._extent.width, device._extent.height );
builder.add_image_usage_flags( VK_IMAGE_USAGE_TRANSFER_DST_BIT );

auto swapchain_ret = builder.build();
if ( !swapchain_ret )
{
    throw Error( swapchain_ret.error(), swapchain_ret.vk_result() );
}
return { device._device, swapchain_ret.value() };
```

That's some fairly classic usage of `vk-bootstrap` that I originally took from
[vkguide](https://vkguide.dev/). We request a fullscreen swapchain with vsync enabled,
using 32 bits sRGB colorspace.

On Windows it works just fine. On Mac it doesn't. Or more precisely, it doesn't,
but the API makes it look like it does.

See, for some historical reason Mac does not use RBGA, they use BGRA<sup>[2](#myfootnote2)</sup>.
So our desired format `VK_FORMAT_R8G8B8A8_SRGB` is unsupported. The correct one we should
be requesting is `VK_FORMAT_B8G8R8A8_SRGB`. It's an easy error to make if you are not
well versed in Mac lore, and would be trivial to spot if `SwapchainBuilder::build()` failed
if the requested format isn't available. But it doesn't.

Instead it does this:

```cpp
VkSurfaceFormatKHR find_best_surface_format(
    std::vector<VkSurfaceFormatKHR> const& available_formats,
    std::vector<VkSurfaceFormatKHR> const& desired_formats)
{
    auto surface_format_ret = detail::find_desired_surface_format(available_formats,
                                                                  desired_formats);
    if (surface_format_ret.has_value()) return surface_format_ret.value();

    // use the first available format as a fallback if any desired formats aren't found
    return available_formats[0];
}
```

It returns the first format supported by the device, queried from
[`vkGetPhysicalDeviceSurfaceFormatsKHR`](https://docs.vulkan.org/refpages/latest/refpages/source/vkGetPhysicalDeviceSurfaceFormatsKHR.html). Worse, the spec makes no guarantee in which order the
formats are returned, so different devices could return the same list different orders, and any
driver update could also break it.

## Explicit over implicit

To quote the great Scott Meyers "Make interfaces easy to use correctly and hard to use incorrectly".
Fallbacks, especially hidden ones like this, make this easy to use incorrectly. Even ignoring
the fact that the fallback here isn't even deterministic, having an implicit fallback at all is bad API
design in my book.

It is still possible to catch the error, as the returned `vkb::Swapchain` object has an `image_format`
member. One could test if the returned swapchain is indeed using the requested format,
but it feels unnatural and clunky. The whole point of the builder API
is that it returns an `expected<Swapchain, Error>`, so a user would naturally expect
that a call returning `Swapchain` did create something that satisfies all the requested settings.

Similarly, some other parts of the API look like they do the right thing, but actually don't.
For example in my project I call this when the user toggles vsync off:

```cpp
builder.set_desired_present_mode( VK_PRESENT_MODE_MAILBOX_KHR );
builder.add_fallback_present_mode( VK_PRESENT_MODE_IMMEDIATE_KHR );
```

While writing this article I realized that despite looking like the fallbacks are made explicit,
`vk-boostrap` will in fact still silently fallback to `VK_PRESENT_MODE_FIFO_KHR` (vsync on) if both
the desired _and_ the fallback present modes are unavailable. Again a user can double check the
`present_mode` member to see which was actually selected, but to me this should only be in case
one of the explicitly requested values was used.

## API design is hard

I don't want my readers to walk away from this article thinking "wow `vk-bootstrap` is bad".
I'm just using a recent interaction to illustrate a broader point.

I think this holds for any domain, but especially in low level and hardware abstraction APIs where
being strict and specific is the point. After all, Vulkan and friends were designed as a reaction
to the more implicit OpenGL and DirectX of the time. Likewise, we use C++ and the like because
we want to be explicit about resource usage. You'd hate to use a container and have it implementation
defined whether the backend you get is `vector`, `deque` or `list`.

And if you are not convinced yet about the necessity of being explicit rather than implicit in API design,
I would like to remind you that fallbacks are de-facto part of the API contract. Meaning that users will start
relying on which fallback they get if their desired format isn't available. I guarantee they
will start filing bug reports if it changes, arguing that _you_ broke their code, even if all you did is pass
along the first thing returned by `vkGetPhysicalDeviceSurfaceFormatsKHR()`.

It took C++ a while to realize that `explicit` should have been the default, but by now we should
all have taken that lesson to heart.

---

<a name="myfootnote1"><sup>1</sup></a>: For those unfamiliar with graphics, swapchain is a queue
of framebuffer textures that your program draws to. Most commonly you draw on frame N while the
window is presenting frame N - 1.

<a name="myfootnote2"><sup>2</sup></a>: The red and blue channels are flipped. Mac started out
as ARGB on big endian PowerPC, which became BGRA on little endian x86 (and now ARM).
