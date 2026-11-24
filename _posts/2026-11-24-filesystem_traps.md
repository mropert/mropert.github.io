---
layout: post
title: "4 filesystem performance traps in the standard library"
tags: [cpp, gamedev]
description: > 
  C++ might claim to be the language of performance, but when it comes to file
  operations its standard library is a bit of a minefield. And it's not
  even (entirely) Windows' fault.
author: mropert
---

For the past 2 years I have been [toying with graphics](/2025/12/30/a_year_with_graphics/) and one
of the things I keep improving is the startup time. Last year I wrote about
[using executors vs TBB](/2025/11/21/trying_out_stdexec/) to throw more cores at the problem.
With that more or less addressed, the bottleneck has moved to the filesystem. Or more
precisely, to misuses of the C++ file and filesystem APIs.

Below are 4 performance traps in the standard library that you need to be aware of.
I have personally had to deal with each of them, although some are older and
date back from my time [working with VFS on Paradox games](/2020/07/26/threading_with_physfs/).

## Trap #1: `std::filesystem::exists()` is almost always redundant

C++20 added `std::map::contains` which I dislike because it solves the wrong problem.
The rationale was "it looks nicer than `map.find( x ) != end( map )`"
(or `map.count( x ) > 0` which is equally terrible in my opinion). The problem is:
you very rarely need to check if a map contains a given key without a follow-up action
that will also require a lookup in the map. So essentially, those APIs increase the
likeliness that one map lookup will become 2 or more. It's one of the very common
performance pitfalls I can find in just about any codebase.

And this is not limited to `map`. It's also in `unordered_map` and `flat_map` and most
3rd party or in-house hash maps out there. And as it turns out, most filesystem libraries
encourage the same pattern.

Accessing a file isn't free. Linux might be pretty good at keeping an inode cache in the kernel,
but it's still costing a filesystem call and a path resolve. And on Windows, any file operation
on an arbitrary path is expensive (especially since there's usually an antivirus callback that gets
injected in the chain). So why bother checking if a file exists, something every filesystem call
already does for free?

I have seen the following in many projects:

```cpp
if ( exists( file ) )
{
    std::ifstream fd( file );
    if ( fd )
    {
        // read some stuff...
    }    
}
```

The second `if` is the big hint here: file open already handles the case where the file doesn't exist. 
There is no point in checking if the file exists beforehand. As I sometime write in code reviews:
"just do the thing".

The same goes for meta file operations, like `rename()` or `stat()`. They will tell you if the file
isn't there when they try to do the thing, it's pointless to check beforehand. Actually, it's
worse than pointless: it's wrong. Because even if `exists()` returns `true`, the subsequent call
might still fail because the file was removed in between the calls.

## Trap #2: Calling `std::filesystem::file_size()` before reading

Checking how large a file is before reading it isn't uncommon. The textbook case
is buffer sizing: we want to allocate some memory to hold the content of the file<sup>[1](#myfootnote1)</sup>.
Another common option is header parsing and validation, where we check the file size against
an entry in its header to ensure it hasn't been truncated (and fail early if it does). 

A common answer that comes up since C++17 is to call `file_size( path )`. But again
as with the previous trap, how likely it is that we will get a file size and then
not open it? Because if we do, there's a cheaper way.

On Linux and Mac, calling `stat()` on a path to get its size isn't horribly more expensive
than seeking to the end, saving the current file offset position, then rolling back
to the start. But on Windows again, the `GetFileAttributesEx()` that hides behind
`std::filesystem::file_size()` is about as expensive as opening the file itself,
which isn't great.

When using `std::ifstream`, consider using `seekg()` and `tellg()` instead of calling
`file_size()` beforehand. And if you don't want to use `<fstream>`
(see Trap #4 as to why you might not), remember that the C API `fseek()` and `ftell` use `long`
and not `size_t`, meaning they will not work on files bigger than 2GB on some platforms.
There is sadly no portable way to get a 64 bits file size that way,
so you will need to rely `#ifdef _WIN32` to use either
[`_fseeki64()`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/fseek-fseeki64) and
[`_ftelli64()`](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/ftell-ftelli64)
or [`fseeko()`](https://man7.org/linux/man-pages/man3/fseek.3p.html)
 and [`ftello()`](https://man7.org/linux/man-pages/man3/ftell.3p.html).

But maybe you also care about querying other file attributes too, in which case...

## Trap #3: Using `std::filesystem::last_write_time()`

Even if we ignore the ability to query metadata on already open files, `std::filesystem`
still provides some questionable abstractions similar to `file_size()`, such as
`last_write_time()` and `status()`.

The issue with all those calls is that they all map to one system call (our friends
`stat()` and `GetFileAttributesEx()`) that returns all the info in one struct.
But as those calls are pure, they have no way to cache the other results, meaning
querying all three will translate into three identical system calls (and discarding
all the other values).

Fortunately, C++17 also came with `std::filesystem::directory_entry` which can be constructed
from a path and as per the specification will update and cache file metadata in bulk
when the platform allows.

So if you find yourself needing any file metadata without needing to open the file, this
is the way to query metadata regardless how many fields you care about.

But what if the file is already opened in a `std::filebuf` or a `std::istream`?
Sadly there is no way to construct a `directory_entry` from those (or even a `FILE*`).

But for those willing to drop down to platform dependent APIs, since C++26 and [p1759](https://wg21.link/p1759)
we can now get the OS handle to an open file through `.native_handle()`.
And then from it call [`fstat()`](https://man7.org/linux/man-pages/man3/fstat.3p.html)
or [`GetFileInformationByHandle()`](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-getfileinformationbyhandle).
This is of questionable value on POSIX, but again significantly faster on Windows as it skips
both path resolution and (most importantly) another antivirus check.

And while we're on the topic of performance on Windows, let's finish with:

## Trap #4: Using `std::ifstream` with MSVC for large reads

I don't have a nicer way to put it: the implementation of `std::ifstream` on MSVC is bad when it comes to large reads.

`ifstream` is supposed to be a buffered file reader abstraction that protects the user from the physical reality of
hard drives (both HDD and SSD). Block devices, as their name implies, perform reads and writes in blocks
(usually 4K, up to 16K and more depending on controller). Reading anything smaller will result in wasted data
that will need to read again later (from kernel disk cache if lucky, from the physical drive if not). To avoid
that, `ifstream` (much like `fread()`) keeps a read buffer attached to the file descriptor and only makes
an actual `read()` filesystem call when the buffer runs out. This is critical for text parsers that read a few
characters at a time.

For large reads that span multiple pages (for example, reading an entire image), the buffer becomes a hindrance
as it breaks down the work into small 4k reads followed by a `memcpy()` to the actual destination. This is
why sane implementations of `fread()` and `std::ifstream` have a test for the read size that bypasses the
internal buffer and reads directly into the destination. This is the case for all implementations of `fread()`
that I know of, and for _most_ implementations of `std::istream`. Meaning it's broken on MSVC.

This is not a new thing. There's a [bug report](https://github.com/microsoft/STL/issues/1107) open since 2020.
I suspect the underlying issue might be older since MSVC guarantees ABI compatibility with objects built 
with VS2015. On my i7 10700 with an M.2 SSD, this makes reads 3 times slower for large files
(I found it in my texture loader).

The solution, sadly, is to drop back to C and use `fread()`. Which probably means rolling out your own
`File` class. On the plus side, it means we can combine it with the previous point and use OS system
calls to query metadata from opened files. The bad part, of course, is that we are now rewriting entire
features that the standard library is supposed to be there for.

## Thoughts on the future

What can we learn from this? To me the important thing to keep in mind is that most of those
could be solved in the standard.

Points 1 and 3 can be fixed by deprecating the metadata query free functions in favour
of only `std::filesystem::directory_entry` (or at least teaching people that those are
inefficient convenience functions, but what happened to "make APIs hard to misuse"?).

Points 2 (and also 3) are a bit more tricky because not all fields from `directory_entry` can be
easily retrieved from a file descriptor (in particular the path on Linux), and the semantics of
`refresh()` could become weird. Maybe we could extract file attributes into a struct, that can then
be retrieved from either `directory_entry` or an opened file handle.

Point 4 is an implementation bug so more standard wouldn't help. Historically the committee
has tried to avoid putting things in the spec that they consider "quality of implementation"
<sup>[2](#myfootnote2)</sup>. I guess the only solution is to complain even more at Microsoft.

Or maybe I should just publish the basic `File` class I had to write in my current project to work
around those issues, and you could copy-paste it into your codebase...

---

<a name="myfootnote1"><sup>1</sup></a>: Some may question the sanity of making a heap allocated
buffer and then reading an entire file into it when `mmap()` exists, to which I answer: predictability.
With `fread()` you know exactly when your thread will block on I/O, with memory-mapped files you
don't. I do not recommend trying it with [collaborative multi threading](https://www.youtube.com/watch?v=hbFJi9G9rsg).

<a name="myfootnote2"><sup>2</sup></a>: The argument being that QoI wording might bar implementation
from adopting a more efficient solution in the future if the wording is too specific.
The counter argument is that C++ standard libraries will basically never do that either way,
because that would be an ABI break and they consider it to be
[more important than performance](https://cor3ntin.github.io/posts/abi/).
