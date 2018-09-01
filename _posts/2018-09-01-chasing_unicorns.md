---
layout: post
title: Chasing Unicorns
tags: [cpp]
description: >
  Are the problems we face really unique? What is the chance that we're the first to encounter them?
  How can we gain insights from others who already solved them? Why aren't we?
  Are unicorns real??!
author: mropert
---

Picture this situation: a new developer joins a team. Sooner or later, he discovers the existing codebase and starts
playing with it. Some time may pass, but at some point he encounters a pattern he finds weird.
Maybe he read a case against it in a book, or heard it in a talk, or simply learned from experience that this is probably
not the recommended way of solving that particular problem.

He brings up the matter to another programmer and points-out that particular construct before suggesting
the different approach. He makes his case but is switfly rebuffed: "we don't do things like that here".
Surprised but not defeated, our developer brings out his sources: it's not just his weird idea, it's
more of a consensus that the industry has reached over the years. And then the hammer falls:
"this doesn't apply to us, we have unique constraints".

He has found a unicorn.

## One in a million

Does this story feel similar? Maybe brings backs a past experience, or perhaps a story shared over a beer?

It usually works as follows: a suggestion to use an engineering best practice is ignored because it can't
possibly apply to this particular project or company. Maybe it comes from another industry, from a company
of a different size, it may even have been tried in the past with unsuccessful results!

In some cases, that reason may hold. Maybe the idea comes from a very specific and different use case and
can't be generalized. But most of the time, what we call best practices are accepted all over
the board.

What are the chances that a given project is, in fact, a pioneer on the domain and has indeed a use case
that reveals a pitfall in an accepted practice? There is certainly a possibility, but one must ask himself
if it really is the one that defies statistics because most of the time, it won't be.

## Take the Unicorn Test

Reading this article, my readers may already have some concrete examples in mind that would be related
to the point I'm making here.

Let's start with one of the most obvious: testing. I believe that today everybody knows that testing is good,
usually the more the better, and done at the lowest possible level. This is not a new idea, in fact
as Kevlin Henney [pointed out at ACCU 2018](https://www.youtube.com/watch?v=mrY6xrWp3Gs), you could
find references to the idea of Test Driven Development in the 1970s. Yet it is not that hard today, more
than 40 years later, to find projects without any unit tests at all.

Another well-known practice is to stay away from `goto` instructions. As Kevlin also pointed out in his
talk, the recommendation came from [Dijkstra himself, in 1968](https://dl.acm.org/citation.cfm?doid=362929.362947).
A 50 years old best practice that still managed to be ignored and produced a 
[critical vulnerability in iOS in 2014](https://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch/)!

Even more concerning, in my opinion, is the fact that while looking for references to the bug I could find recent articles
defending the use of `goto`, stating that the problem was somewhere else. A
[2015 paper](https://peerj.com/preprints/826v1/) even analyzed Github projects
and concluded that "developers limit themselves to using goto appropriately in most cases".
Fortunately the C++ Core Guidelines still stand by Dijkstra stating "ES.76: Avoid `goto`".

## Most common C++ unicorns

(This section will delve into C++ specific practices, if you, dear reader, isn't interested in C++, first I'd like to
thank you for reading my blog anyway, that must be a difficult experience, and second, suggest you to skip to the next one)

As I mentioned in the previous paragraph, the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
are a formidable source of best practices in C++ today. They may not the best way to learn (books, talks, articles and conferences
are a better vehicle for that), but when in doubt, it's a pretty solid go-to reference (pun intended).

To help my readers figure out if they may be working on an unicorn C++ project, I'll list a few best practices that
I found or heard to be discouraged in some places:

* Using the STL. That may look obvious, but some still don't trust it today. This may come from previous bad experiences
  (some implementations were indeed buggy in the 90s/2000s), a policy against templates (see next item) or simply a case
  of NIH ([Not Invented Here](https://en.wikipedia.org/wiki/Not_invented_here)). Whatever the reason, the fact remains
  that the STL today is well designed library provided with every compiler on every platform. Yes, some bits have their shortcomings
  (iostreams, futures...), and some containers could be better optimized if some constraints from the standard were
  to be relaxed (`std::unordered_map` for example), but in which case there's probably a replacement or a more specialized
  alternative in discussion somewhere. And if not, it's probably a prime subject for a new paper. You may use alternatives
  to some parts of the STL if you know what you're doing, but not using it at all makes little sense.

* Using templates. Once upon a time, compilers were having a hard time with two-phase name lookup, or SFINAE, or inlining.
  I had to deal with a linker that did not eliminate duplicate instantiations of `std::string` across translation units,
  leading to ridiculous binary sizes.
  But that time is beyond us. Today GCC, Clang and MSVC will do wonders, allowing us to write generic containers and algorithms
  that can be optimized better than assembly code written by humans.

* Using [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization). The idea is more 30 year old now.
  This is the reason why C++ doesn't need `goto`, or manual use of `new` and `delete`. This is why we should use smart pointers.
  This is how we can have extremely performant native code without resource leaks or garbage collection.
  Scope-based management of resources is in my opinion one of the "killer" features of C++ that most languages lack.

* Avoiding raw loops. C++ algorithms are fast, have names that express intent, and will ensure that cases with only 0 or 1 elements
  in a collection still work. If you don't know what `std::rotate()` does by heart, you probably haven't watched
  [my favourite talk of all time](https://www.youtube.com/watch?v=IzNtM038JuI) by Sean Parent. It even contains a short
  story about how he was told that "we don't use `std::rotate()` here`" and what impacts it had on the product.

* Prefering values over pointers. Pointers are useful, but their semantics are weird. They can be null, copying them doesn't
  really copy anything, pointer data members don't enforce `const` correctness the same way as values, and at some point
  they may refer to freed memory. Pointers to arrays are even worse as they don't carry information about boundaries.
  Herb Sutter made a good case against storing pointers in his [CppCon 2015 keynote](https://www.youtube.com/watch?v=hEx5DNLWGgA).

* Using `auto`. Try a little contest with your compiler: pick any number of C++ expressions and try to find out their type.
  Guess which one will be 100% correct and which one will miss a few corner cases. Spoiler warning: the compiler wins.
  In fact, Bjarne himself wanted a similar mechanism in the early stages of the design of C++. In C (up until C99) you could declare a variable
  without a type and it would default to `int`. C++ wanted to change that to whatever is the type of the right-hand side expression
  but couldn't because of backward compatibility. In 2011 we finally got `auto`, the magical identifier that is always of the right
  type and raises an error if not initialized at declaration. Yes it doesn't implicitly propagate `const`, `&` and `&&`, and for a
  good reason: you want those intents to be explicitly expressed. 
  Again, [Herb's argument](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/) in its favor is 5 years old now.

* Prefering C++ abstractions over C (or assembly). A generalization of some of the previous tropes cited before, some projects
  may still argue that C is simpler and closer to the metal than C++, hence better or faster. GCC's own developers had to fight
  for [the right to use C++ over C](https://lwn.net/Articles/542457/). One the reason LLVM/Clang exists today is because GCC's C
  codeline was impossible to work with at the time. For those still worried that higher level abstractions mean lower performance,
  I'd recommend Jason Turner's [CppCon 2016 keynote](https://www.youtube.com/watch?v=zBkNBP00wJE) and Matt Godbolt's
  [CppCon 2017 keynote](https://www.youtube.com/watch?v=bSkpMdDe4g4).

## If you're arguing, you're loosing

The reason I wrote this article is mostly to help my readers recognize that they might be working on a unicorn project.
One of the big thing I get from conferences is the realization that my problems are in fact, not unique at all and
shared by other people who may have solved them, or may be stuck by the same dogmas I've been facing.
I do believe that the simple fact that you can find someone else from the profession with the same issue
helps dispelling the unicorn's myth.

Still knowing you're facing a unicorn is only half the battle, you then probably want to slay the beast.
I found that most of those antipatterns are linked to the project's (or company's) culture and are not easy to change.
Academic research has found that humans usually have a strong bias against ideas from the "outside".

I've linked some good papers and talks defending those practices and their history in this article, and you can certainly find a hundred more
freely on the internet. On a more generic topic, Dan Saks made a [brillant keynote at CppCon 2016](https://www.youtube.com/watch?v=D7Sd8A6_fYU)
about changing people's mind on a given practice. I fear that trying to summarize it here will not do it justice,
so I strongly suggest my readers to go and watch it.

In conclusion, this time again I encourage my readers to go to conferences and meetups, talk to their peers, watch more talks and read
more articles. For every problem you face, chances are someone else has already faced it and the profession has found a good
solution to it. Remember: unicorns aren't real, like most mythical creatures, they're cool because they can't exist.
