---
layout: post
title: About leaky abstractions
description: >
  There is one kind of leak that neither RAII nor garbage collection can fix, it's abstraction leaks.
  The idea was coined by Joel Spolsky back in 2002 and remains one of my favourite computer science article.
author: mropert
---

[The law of leaky abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/)
by Joel Spolsky is one of the first technical article I encountered after I got my engineering degree
and it remains one of the most influential to me.

It's been published almost 15 ago years and it's well past due time I shared it with you in case you missed it.

## What's a abstraction?

Abstraction if one of the most fundamental concept in computer science.
As the saying goes: "Any problem in computer science can be solved by adding another layer of indirection".

The idea is quite simple: you decompose your problem in layers in which each layer handles a separate concern.
Every layer as a clear goal and focus, which can be reasoned about independently of the rest.
This reduce coupling, enables testing and of course allows for better portability.

For example, when you write C++, your machine cannot understand it. So you use a compiler to translate that code
to assembly. But assembly itself is just a text representation of machine binary code, so an assembler is needed
to translate that. Now say you are using a x86 machine like the laptop I'm using to write this article,
your CPU doesn't even operate with the x86 CISC instruction set at its core. Instead, it translates those
instructions into simpler ones and dispatch them to RISC pipelines. Then, those are again just an abstraction
of basic logical OR, AND, XOR and NOT operations. Which in turn are abstractions build upon transistors
that combine two electrical inputs into one.

You can find similar constructs in any part of computer science, such as networking, filesystems, graphics, you name it.

## Leaky abstractions

In his article, Joel Spolsky postulates that all abstractions are, to some degree, leaky.

What it means is that perfect abstractions do not exist and some quirks do show up from time to time: one layer leaks
into another. Let's take a simple example:

```cpp
std::ifstream ifs("/some/file.txt");
std::string text(std::istream_iterator(ifs), std::istream_iterator());
```

This simple code reads the content of text file `/some/file.txt` into a `std::string` called `text`. It uses the
STL's abstraction of file streams which in turn relies on the operating system file i/o abstraction.

In a perfect world, the owner of this code wouldn't have to think about what `std::ifstream` does. It would behave
the same way whatever the circumstance. But a filesystem is an abstraction of various storage devices and so
reading a file could mean fetching blocks from a physical hard drive, a RAID disk, a network drive, a memory image
or even a floppy disk<sup>1</sup>. And depending on that, your access time will differ by possibly an order of magnitude.

Let's say you have to read and crunch some data from many files. If your CPU has multiple cores, it might make sense
to have a several threads processing different files at once. If a filesystem abstraction was perfect, you could
simply write an algorithm that does just that. But in practice, depending on the storage of each file, it might be
better to have only one reader thread (at the risk of starving your calculation threads) or to have each thread
read and compute their files (at the risk of I/O contention).

Another easy example is this:

```cpp
int sumXY(int* array, int X, int Y) {
  int sum = 0;
  for (int x = 0; x < X; ++x)
      for (int y = 0; y < Y; ++y)
        sum += array[y * X + x];
  return sum;
}

int sumYX(int* array, int X, int Y) {
  int sum = 0;
  for (int y = 0; y < Y; ++y)
    for (int x = 0; x < X; ++x)
        sum += array[y * X + x];
  return sum;
}
```

If you ask a mathematician, he will answer that those two functions are exactly the same.
Indeed, summing by column or by row doesn't change anything in mathematical terms.

But what does the reality says? [sumXY is much faster that sumYX](http://quick-bench.com/BAinb6PP8QaQAnrgbvbBdswLFjs).
Why is that? Because a C++ program is an imperfect abstraction of how a real machine work, and does not account
for [locality of reference](https://en.wikipedia.org/wiki/Locality_of_reference) which make array traversal
by row much faster than by column.

## Corollary

Since all non-trivial abstractions are imperfect and will leak in one way or another, this means any serious programmer
should always have at least a minimal knowledge of the abstractions he uses. This usually means knowing a bit about how
your CPU works, how your network protocols work, how your kernel works and so on and so forth.

I've seen [talks about Javascript that stop midway to explain how x86 protected mode work](https://www.destroyallsoftware.com/talks/the-birth-and-death-of-javascript),
I've worked with financial application engineers that worked on wifi routers on weekends and I'm pretty convinced
this is the way to go.

So don't be shy. On the contrary, be curious. Tear² the thing you're using the most down and see
how it works behinds the scene. It'll help you one day or another, trust me.
