---
layout: post
title: "Benchmarking with Vulkan, or the curse of variable GPU clock rates"
tags: [cpp, graphics]
description: > 
  Trying to get reliable benchmarks on a GPU that keeps adapting its clock rate.
author: mropert
---

Choosing between two implementation often requires answering the age-old question "which is faster?".
Which means measuring/benchmarking. Now what do you do when your device's default mode of operation
gives you unreliable numbers?

While modern CPUs have dynamic frequency scaling with technologies like TurboBoost, in my experience this
hasn't been a huge deal for comparing two benchmarks (as long as you handle P vs E cores). GPUs on the other
hand are bit more capricious. According to GPU-Z, my RTX 2800 is currently running a 300MHz on the GPU and 100 MHz
on the VRAM while I'm typing this article. This is obviously not its usual frequency under moderate or heavy workload.
According to the internet the GPU should run between 1650 and 1815 MHz, and the VRAM at about 1937 MHz.
The numbers are off by a factor of 5-6 on the GPU and 19 on the VRAM. That's quite the discrepancy.

## Steady measurements

This mechanism of dynamic frequency scaling is neat because is saves on power draw, puts less stress on the hardware
and lower the decibels created by the cooling fans. But it sucks for benchmarking.

On my current project I was trying to compare 2 ways of rendering meshes by having a simple toggle in the UI that would
select which shader is used and keep a basic average of the last 60 frames for each mode. But I kept getting nonsense.
More precisely I got the occasional weird jitter. The scene would take 2ms for a while, then suddenly jump to 4 or 6ms,
before going back to 2ms or sometimes even less.

This is not a new problem and it's somewhat well documented that GPU benchmarks should be done with fixed/steady clocks.
But I admit I thought that if I would just disable vsync and use `VK_PRESENT_MODE_MAILBOX_KHR` it would keep my GPU
busy enough to not throttle down much. Sadly this wasn't what I observed.

## The good, the bad and the ugly workaround

A common recommendation I've seen online is to run `SetStablePowerState.exe` a simple exe you can build from source (or download)
that was once [provided by Nvidia](https://developer.nvidia.com/setstablepowerstateexe-%20disabling%20-gpu-boost-windows-10-getting-more-deterministic-timestamp-queries).
What it does is create a DX12 device and call the developer/debug API function
[`ID3D12Device::SetStablePowerState`](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-setstablepowerstate)
which will fix the clock to steady rate until you close it.

It works and it does the trick, but it's been kind of disowned by the company since. The 
[new recommended way](https://developer.nvidia.com/blog/advanced-api-performance-setstablepowerstate/) 
is to use the command line tool `nvidia-smi` to fix the clocks to a desired rate.

I found both to be lacking in some way:
* While `SetStablePowerState.exe` does the job and is simple enough, it is still an exe I have to remember to launch, and close
  when I'm done doing GPU work (or at least benchmarking). If I forget to run it I'll get the wrong results. If I forget to close
  it I'll leave my GPU running at max clock all night.
* `nvidia-smi` is even worse in my book. First it doesn't automatically pick a clock speed for me. The recommendation is to run
  `SetStablePowerState.exe`, then look up the clock values in GPU-Z or similar, note those down, _then_ invoke `nvidia-smi` with the
  right numbers to fix the clocks. Worse, unlike `SetStablePowerState.exe` it doesn't stop if you close the window. There is no window
  to close. You have to invoke it again, once with `--reset-gpu-clocks` and another with `-reset-memory-clocks` to get back to
  the default behaviour. If I would probably remember to close `SetStablePowerState.exe` most of the time, I would very likely forget
  to run `nvidia-smi` and eat up my hardware's lifetime.

And so I went for the third option, make my own utility.

## A simple API

All I wanted was simple: the clocks should be fixed while I run my benchmark or comparison scenario, and off again when it's done.
If my renderer library was based on DX12 it would be easy, just call `ID3D12Device::SetStablePowerState()`,
but sadly Vulkan as no such equivalent (there is an [extension request](https://github.com/KhronosGroup/Vulkan-Docs/issues/2101)
but it doesn't seem to be getting much traction).

But as it turns out, nothing stops you from creating a DX12 device context in a Vulkan app. So I
[did just that](https://github.com/mropert/gpu_stable_power).

My API is quite simple:

```cpp
#include <gpu_stable_power/gpu_stable_power.h>

int main()
{
    // Defaults to off
    gpu_stable_power::Context stable_power;

    // Lock clock speeds
    stable_power.set_enabled( true );

    // Do benchmark

    // Optional: manual toggle off
    stable_power.set_enabled( false );

    // Automatically disables itself on destruction
}
```

The way I use it is I keep it off by default, but I have a toggle in my debug UI to activate it when I need to benchmark.
That way it's always available when I need it, and I cannot forget to turn it off since it's at worse disabled when my app exits.

Implementation wise, it's very close to `SetStablePowerState.exe`. Create a DX12 device for adapter 0 and call
`ID3D12Device::SetStablePowerState()` when toggled on/off. The rest is mostly there to make integration less painful.
The implementation is hidden behind a pimpl (so you don't get DirectX SDK included in a header file), and it turns
into a no-op on non Windows builds (for portability) and release builds (since this API is locked behind Windows 10/11 developer mode).
The criteria can be overridden by setting `GPU_STABLE_POWER_ENABLED` in the library's build settings.
And if you hate CMake, you can just add `gpu_stable_power.cpp` to your build. Finally I've used `#pragma comment lib` to
add `DXGI.lib` and `D3D12.lib` to the linker when needed and keep the build integration to a minimum.

I have not bothered adding GPU selection because I only have one, and I don't have the hardware available to test if it
should be disabled for other vendors (I assume AMD also has variable clock rates?), but it should be easy to add if
the need arises.

That's it for today, happy benchmarking!
