---
layout: post
title: "An Honest Review of AI Programming"
tags: [cpp, gamedev]
description: > 
  They gave me a Claude subscription and told me to get tokenmaxxing, so I tried
  to give it a fair shot.
author: mropert
---

It's getting hard to avoid LLMs these days. Even if you avoid social media (or at least curate
a follow list that avoids the bulk of the slop factory) and shrug off grandiose marketing statements
that end up taken at face value in the news, it will likely come and find you at your place of work.
Unlike the silver bullets of the past (like microservices or NoSQL), AI adoption seems to have been
mandated in many places from the top layer of management, regardless how many of them ever worked
(or studied for) an engineering job.

I do admit that this approach immediately triggered my contrarian side and made very defiant of any
AI tool. I don't believe someone who has never written a line of code in their life should be telling me
what to use for my engineering job. This sounds to me like the most terminal case of micro-management,
and that's never a good thing (on top of being personally insulting).

Either way, over the past 3 months I got to use Claude and friends for work and I have to admit I found
it somewhat useful. As long as you don't ask it to write code. Please don't ask it to write code.
But I'm getting ahead of myself

## Artificial "Intelligence"

You probably heard this a million time by now, but artificial intelligence really isn't that intelligent.
It's all marketing and buzzwords. All we really we have here is a (very) large neural network specialized
in natural language processing.

As it turns out a lot of what we humans do on the computer is use text to communicate both ways, that model
can be used to parse queries, generate textual<sup>[1](#myfootnote1)</sup> responses based on a probabilistic
heuristic or generate command lines that can be then executed the old fashioned way and their results fed back
into the model to make a loop until we reach some exit condition.

That's not to say this is inherently bad. But it's not magical either. "Agentic workflow" (or whatever they're
calling it at the time you're reading this article) is just the realization that every software problem can be
solved by adding another layer of indirection, and LLMs are no exception. If the neural network output can
be improved by providing more input, then attach more input. And if the best way to figure out which input
that would be is to query the model to generate a command and then pipe it through `system()`, so be it.

But let's focus on the user point of view for the rest of this article.

## We have Google at home...

There's tons of valuable stuff out there on the internet. And assuming we do a decent job at keeping existing
records intact, the net total sum of public knowledge can only improve. The problem is finding it. This isn't
a new thing. I'm old enough to remember the time when you'd first get suggested to try this new "google" thing.
But sadly it's gotten quite worse since the 2000s golden age. They have been fighting an uphill battle against
SEO for a while now, and it doesn't look like they're winning<sup>[2](#myfootnote2)</sup>.

Enter AI, as both a help and a hindrance. As a search tool, I have found it usually good as answering pointed
questions expressed in natural language, especially given the conversational ability that allows refining the answer
or bring up follow-up questions with the current context in mind. In practice it is the equivalent of running
a bunch of searches, skimming through the top N links, and repeating until we think we've got the picture.
Writing a short summary of a longer texts seems to be what LLMs are best at, and so it makes sense to use it to
automate the process. Plus crawling and summarizing multiple searches is an inherently parallel job so it isn't
hard to see the efficiency that can be brought up by automating the process, assuming we have enough compute
available and it doesn't hallucinate the summaries, we'll get back to both those points later because they are
quite important.

The hindrance counterpart is of course that LLMs have lowered the cost of flooding the internet with word salad
that dilutes an already precarious sea of information. Freya Holmér published a [very good video essay](https://www.youtube.com/watch?v=-opBifFfsMY)
on the topic and impact of littering the web with AI generated content in the never ending SEO arms race.
Searching for information by hand is still an option today, especially if you already know reputable sources on a
particular topic. But if you're not, then a lot of care must be taken to sort out the slop from the real data,
and automating it with an LLM loop may not help if you can't instruct it which sources to keep and which to discard.
Even before we talk about hallucinations, an LLM generated summary can only be as good as the sources it ingested.

## ...and now we also have Google at work

A particular area where I found an LLM loop useful for searching, that I haven't found much discussion about so far,
is internal company knowledge bases. Wherever you work, chances is there is a combination of wikis, slack conversation,
Google Drives, Confluence pages and whatnot where a lot of good information is sitting but no one ever seems able to
find it. I've experienced some of those "enterprise solutions" to be so bad that sometimes I couldn't find a page
I browsed a week before even when I typed the title (or what I remembered the title to be) in the search bar. I have
written my share of internal technical article that I'm fairly sure no one ever found since unless they kept a bookmark from
the time I first linked it in the tech group chat.

In the past few months and despite being new to the company I have been able to find a bunch of answer that were written
before my time, because the same way "AI Google Search" can run multiple queries in parallel and summarize the answer, Claude
and friends can turn my natural language question into a bunch of search queries for likely synonyms until they hit something.
It proved really handy when I ran into a particular edge case with the engine and I could quickly search if someone
ever reported it, offered a workaround or discussed why it had to behave this way. Every time you run into an issue
in tech, chances are it's been discussed before and you're off a much better start if you can find that conversation.

This is technically not new tech. One of the things that has made Google and friends efficient for so long is that they
automatically generate synonyms when building keyword metadata for a page. Your company wiki or chat search functions
likely do not<sup>[3](#myfootnote3)</sup>. I suspect there a lot of value sitting in year old slack conversation logs
that are absurdly hard to access without an LLM to search for them, or a more senior coworker being able to remember it
and point you towards it.

Now is it efficient from a tech perspective to run an expensive LLM to search through a company wiki when they could instead
implement basic search engine techniques that are decades old at this point? No, I'm fairly sure it is not. Indexing
the content the way Google did 20 years ago with would definitely be a much more efficient solution, compute wise.
But from a user perspective it is much more desirable than trying to search for "UI" and getting no results
because the plaintext reads "User Interface" in the page they are looking for.

## Hallucinations and false positives

Once an LLM found an answer, it's usually a good practice to good look at the primary source. Reading that article, document or
chat log will help ensure the answer wasn't hallucinated.

Hallucinations are an inherent property of how LLMs work. Token generation isn't based on logic or truth, it's based on statistics
and from my understanding there is no way to entirely avoid them. I have experienced them on all models, from the cheap (and pretty bad)
free version of Copilot that comes with Bing to the fanciest paid models of Claude.

I have found them most common when asking a very specific question about "how to do X in Y", like asking for a particular niche
feature in CMake or Vulkan or Xcode. Instead of answering "no it can't be done", I got the most probabilistic answer which suggested
I try to click a button that wasn't there or enable a feature flag that wasn't there.

The whole thing is akin to asking someone who's generally knowledgeable about a problem domain but not the specific software or library
you are using and would reply "yeah that sounds like a thing you should be able to do" because it feels like a reasonable expectation
to have and maybe is a feature in similar products. I suspect it's the same reason why when writing code an LLM would try to call a
non-existing API, because based on other languages is sounds like C++ containers should have a `.sort()` function because it's the kind
of thing you can do in Python and C#. That's not the case because C++ make a strong distinction between containers, iterators and algorithms,
but LLMs do not reason (despite their marketing calling it "reasoning") from first principles.

---

<a name="myfootnote1"><sup>1</sup></a>: I know that LLMs can also generate image and video formats. But it's 
out of scope for the purpose of this article. Also the results suck. AI "art" is a non-starter in my book.

<a name="myfootnote2"><sup>2</sup></a>: If you like cooking and sometimes search online for recipes,
you know exactly what I mean.

<a name="myfootnote3"><sup>3</sup></a>: Email on the other hand has seen some progress since the time where you had to install Google Desktop to be
able to find anything in Outlook, probably because the biggest email providers today also run a search engine and an AI research
division.
