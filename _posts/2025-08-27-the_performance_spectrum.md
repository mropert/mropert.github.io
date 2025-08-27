---
layout: post
title: "The Performance Spectrum"
tags: [cpp, gamedev]
description: > 
  We take a little break from our "What Makes A Game Tick" series to get a little bit philosophical about the topic
  of performance and what it means for different people.
author: mropert
---

I was at [CppNorth](https://cppnorth.ca/) a month ago and after listening to a few talks and having conversations with fellow speakers
and attendees I realized that despite all of us being here because we care about performance to some degree (else we'd all be doing Python or Javascript or something)
when it came to nitty-gritty details, I wasn't sure we were talking about the same thing. And maybe that could explain some of the
disconnect I can sometimes observe between various industries talking about using C++ on social media?

At some point after I delivered my talk ([Heaps Don’t Lie - Guidelines for Memory Allocation in C++](https://www.youtube.com/watch?v=74WOvgGsyxs))
I wondered who was my target audience. In particular, I wasn't so sure my past self would have cared so much for it. I hope they'd have
found the talk entertaining but I'm not certain they would have found it super applicable in their day to day business. And here's the catch:
if you asked people I worked with (like I was asked during interviews), they would absolutely have told you they cared about performance. It was fintech after all.
They wrote it all in C (and later C++) to be fast. There was also some Java in there, but the core business functions handling financial models,
transactions and portfolios had to be C++.

And so, it got me thinking, are some of us engineers dismissing each other's thoughts and approaches because it looks like the other party
"doesn't care about performance" based on wildly different contexts or assumptions?

(Reader warning: I have been mulling over those thoughts for a while now, this article might not be as clear focused as the other ones I made this year)

## When is it too slow?

Since my years in games I have the habit to always keep Optick in my builds, even in toy projects, so I can quickly attach and look for
an odd millisecond in a profiling capture. 60 frames per second gives you 16.67ms to do stuff, so anything takes in the hundreds of microseconds
to a couple milliseconds is suspicious and should immediately trigger the age old "should it really take that long?" question in one's head.

When I worked on a financial pricer, any input from a trader or sales should have responded in about a second or so
(think filling a cell in a spreadsheet with some formulas). A few seconds and you might want to add a spinner to your UI.
We could get away with taking tens of seconds loading up a portfolio and displaying the balance,
although I suspect it did annoy some clients (it definitely annoyed me when it came up in bug reproduction steps).

And even before that, I was working on banking web frontend, but still in C. Yup you got that right, we'd do server side HTML generation
in C. Was it performant? By gamedev standard, absolutely not. For example our dynamic string function would realloc on most appends (no geometric growth,
don't do this at home, as I explained in my most recent talk). But compared to what the J2EE stuff of the day, it was another world.
We could handle peak traffic with maybe 6 servers, and two of them were only there so we could take them down for maintenance and still
be able to run production just fine. We had someone pitch us a Java solution replacement once and it would have required about 10 times the servers.

So when is it too slow? As Tony Van Eerd [reminded us at CppNorth](https://cppnorth2025.sched.com/event/21xRf), the answer, as always, is "it depends".

## The micro benchmark answer

Let's start with the purely technical answers. The most common thing I see argued about on social media is something along the lines of
"it's too slow compared to how I could have implemented it". Or sometimes more charitably "it's too slow compared to the commonly accepted benchmark".
One may argue that given how the algorithm works on today's Twitter we are unfortunately bombarded by the loudest voices and the hottest takes
because they drive "engagement", and so we shouldn't even bother to answer them. I agree, but since this is a conversation I also had offline on the
occasion, I might as well gives some elements of response.

A very easy metric of comparison would be "is it slower than if you had used a readily available library in your codebase?". This could be the STL,
Boost or whatever lies under `common/` or `utils/` in your engine or framework. That case is probably a no-brainer. The best code is no
code, so why add more LOCs that will need to be maintained when you can `#include` something that does the job
(or `import` for the 3 of you who have figured out Modules).

Now, say you run accross a presentation showing something that's not slower than the STL or Boost but is quite slower than something you have in house,
or that you could implement. That's amazing, but if I may ask you, imaginary Twitter poster, why not direct that energy into a pull request?
I'm not gonna ask anyone to make a standard's proposal, I understand and respect the impossible amount of time and sanity it takes to get anything there.
But what about pushing something to GitHub? After all a lot of great libraries (including bits that eventually made their way to the STL)
started as someone putting some code available for download out there and telling people "check out this thing I've done".
This is by the way a great way to get a free trip to a tech conference as a speaker.

But even when presented with a readily available faster alternative, sometimes folks might not bother.
Maybe they don't care about performance after all. Or maybe it's something else?

## The profiler's answer

Calling a virtual method has a cost. It involves a double indirection (look up the vtable, then do an indirect call to the function pointer in it).
A C-style function pointer would be faster. Some argue a `switch`/`case` on an enum followed by a hardcoded call to the concrete implementation is even better.
The whole routing and implementation could possibly be inlined even!

Likewise, using `std::map` might appear convenient but it's quite slow compared to `std::unordered_map`, and then we can start arguing whether Robin Hood hashing
is still the best map/hash table implementation to not be found in the STL.

But what if our virtual method is there to dispatch between DirectX12 and Vulkan? Or between MySQL and PostgreSQL? What if our map is there to represent
a user's configuration file? Do we still care how fast it could go if implemented differently? As the paragraph title hinted at, would it make a noticeable
difference in the operation we are profiling?

When optimizing, one should always go for the biggest bang for their buck. When issuing a bunch of draw calls to a graphics API, the `virtual` calls
are unlikely to be more than a rounding error compared to memory mapping buffers too often or switching shaders between each draw. Likewise,
there might be some gains to be achieved depending how we decide to parse a JSON input, but this might be inconsequential if said JSON is a few dozen
kilobytes returned from a remote HTTP call over the Atlantic.

This may sound obvious, but context matters, and it doesn't always properly accompany the code that's being presented. Even in a long form
like a talk (or a post like this), there's often a bunch of context from the author's background or project that isn't spelled out.
This is both a suggestion to readers and watchers to give the benefit of the doubt, and to authors (and I include myself in this) to
remember to give some frame of reference when talking about speed and performance.

Unless we don't care about performance. Or maybe we do still, but there's something else we haven't considered?

## The manager's answer

An expected but sometimes dreaded answer when an engineer brings up that something could be done differently for better results is
"how long will it take?". Usually followed by "how much better would that make it?". This allows for some quick project manager math
to lead to a "yes" or "no", or maybe an "ok but don't spend more than a day on this".

As much as some of us would like to ignore it, most code isn't written in a world of infinite resources (be it time or money, both interchangeable
for the purpose of this discussion). Worst, different projects have different amount of resources they could be willing to spend
to solve the exact same problem. Google might answer that just about any saving on their core C++ code is worth it because it will be multiplied
by the tens of thousands of server costs this will run on. Someone at AWS once told me they'd only bother with C++ over Java for
problems that were latency bound, because for everything else they could just spin more instances and forget about it.

One can beat the STL's speed by making their own ad-hoc container. Which can sometimes be beaten by hand writing some assembly. But how
much more will it cost to implement? And then a good manager should ask how much it will cost to maintain? Not just in case
of bugs, but what if needs to reused for another case? Needs more functionality? What if the original author left the company?
Will the handwritten assembly written today still beat the C++ compiler optimized version in 5 years? 10 years? What happens when
the product needs to move from x86 to ARM?

When gazing at the improvements FFMPEG can make to a given software decoder by using assembly, let's not forget that the project is still 90% C.
Because there's no money in hand writing assembly code when writing an interface to abstract away dozens of codecs behind one API.
I'd even argue that using C over C++ means a bunch of `switch`/`case` and function pointers that could be done away with a codec base class
and some `virtual` methods.

I can absolutely sympathize with the disappointment of being told to use a pre-made "good enough" solution that could be outperformed
easily (or maybe not so easily but would be a fun challenge). But sometimes code cannot argue with the unstoppable cold force of economics.
Or as Titus Winters nicely puts it "engineering is programming integrated over time".

## Performance is a spectrum

So if you're ever tempted to look at some post or presentation or language feature proposal and think
"those numbers are insane, those people clearly don't care about performance", I hope you can remember this discussion.
And as promised at the top, we will be back to our regularly scheduled program next post for more of "What Makes A Game Tick". See you there!

Oh and here's a reminder that I'm available for [consulting](/consulting) for all your C++ performance needs, including training
your engineers on the performance mindset!
