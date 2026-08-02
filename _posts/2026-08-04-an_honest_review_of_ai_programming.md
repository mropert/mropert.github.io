---
layout: post
title: "An Honest Review of AI Programming"
tags: [cpp, gamedev]
description: > 
  They gave me a Claude subscription and told me to get tokenmaxxing, so I tried
  to give it a fair shot.
author: mropert
customjs:
 - https://platform.twitter.com/widgets.js
---

It's getting hard to avoid LLMs these days. Even if you avoid social media (or at least curate
a follow list that avoids the bulk of the slop factory) and shrug off grandiose marketing statements
that end up taken at face value in the news, it will likely come and find you at your place of work.
Unlike the silver bullets of the past (like microservices or NoSQL), AI adoption seems to have been
mandated in many places from the top layer of management, regardless of how many of them ever worked
(or studied for) an engineering job.

I do admit that this approach immediately triggered my contrarian side and made me very defiant of any
AI tool. I don't believe someone who has never written a line of code in their life should be telling me
what to use for my engineering job. This sounds to me like the most terminal case of micro-management,
and that's never a good thing (on top of being personally insulting).

Either way, over the past 3 months I got to use Claude and friends for work and I have to admit I found
it somewhat useful. As long as you don't ask it to write code. Please don't ask it to write code.
But I'm getting ahead of myself.

## Artificial "Intelligence"

You probably heard this a million time by now, but artificial intelligence really isn't that intelligent.
It's all marketing and buzzwords. All we really we have here is a (very) large neural network specialized
in natural language processing.

As it turns out a lot of what we humans do on the computer is use text to communicate both ways, that model
can be used to parse queries, generate textual<sup>[1](#myfootnote1)</sup> responses based on a probabilistic
heuristic or output command lines that can be then executed the old fashioned way and their results fed back
into the model to make a loop until we reach some exit condition.

That's not to say this is inherently bad. But it's not magical either. "Agentic workflow" (or whatever they're
calling it at the time you're reading this article) is just the realization that every software problem can be
solved by adding another layer of indirection, and LLMs are no exception. If the neural network output can
be improved by providing more input, then attach more input. And if the best way to figure out which input
that would be is to query the model to generate a command and then pipe it through `system()`, so be it.

But let's focus on the user point of view for the rest of this article.

## We have Google at home...

There's tons of valuable stuff out there on the internet. And assuming we do a decent job at keeping existing
records intact, the net total sum of public knowledge can only improve (but, spoiler warning, there's a caveat).
The problem is finding it. This isn't a new thing. I'm old enough to remember the time when you'd first get
suggested to try this new "google" thing. But sadly it's gotten quite worse since the 2000s golden age.
They have been fighting an uphill battle against SEO for a while now, and it doesn't look
like they're winning<sup>[2](#myfootnote2)</sup>.

Enter AI, as both a help and a hindrance. As a search tool, I have found it usually good at answering pointed
questions expressed in natural language, especially given the conversational ability that allows refining the answer
or bring up follow-up questions with the current context in mind. In practice it is the equivalent of running
a bunch of searches, skimming through the top N links, and repeating until we think we've got the picture.
Writing a short summary of a longer texts seems to be what LLMs are best at, and so it makes sense to use it to
automate the process. Plus crawling and summarizing multiple searches is an inherently parallel job so it isn't
hard to see the efficiency that can be brought up by automating the process, assuming we have enough compute
available and it doesn't hallucinate the summaries, we'll get back to both those points later because they are
quite important.

The hindrance counterpart is of course that LLMs have lowered the cost of flooding the internet with word salad
that dilutes an already precarious sea of information.
Freya Holmér published a [very good video](https://www.youtube.com/watch?v=-opBifFfsMY)
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
written my share of internal technical article that I'm fairly sure no one ever found since, unless they kept a bookmark from
the time I first linked it in the tech group chat.

In the past few months, and despite being new to the company, I have been able to find a bunch of answers that were written
before my time. Because the same way "AI Google Search" can run multiple queries in parallel and summarize the answer, Claude
and friends can turn my natural language question into a bunch of search queries for likely synonyms until they hit something.
It proved really handy when I ran into a particular edge case with the engine and I could quickly search if someone
ever reported it, offered a workaround or discussed why it had to behave this way. Every time you run into an issue
in tech, chances are it's been discussed before and you're off a much better start if you can find that conversation.

This is technically not new tech. One of the things that has made Google and friends efficient for so long is that they
automatically generate synonyms when building keyword metadata for a page. Your company wiki or chat search functions
likely do not<sup>[3](#myfootnote3)</sup>. I suspect there is a lot of value sitting in year old slack conversation logs
that are absurdly hard to access without an LLM to search for them, or a more veteran coworker being able to remember it
and point you towards it.

Now is it efficient from a tech perspective to run an expensive LLM to search through a company wiki when they could instead
implement basic search engine techniques that are decades old at this point? No, I'm fairly sure it is not. Indexing
the content the way Google did 20 years ago with would definitely be a much more efficient solution, compute wise.
But from a user perspective it is much more desirable than trying to search for "UI" and getting no results
because the plaintext reads "User Interface" in the page they are looking for.

## Hallucinations and false positives

Once an LLM has found an answer, it's usually a good practice to go look at the primary source. Reading that article,
document or chat log will help ensure the answer wasn't hallucinated.

Hallucinations are an inherent property of how LLMs work. Token generation isn't based on logic or truth, it's based
on statistics and from my understanding there is no way to entirely avoid them. I have experienced them on all models,
from the cheap (and pretty bad) free version of Copilot that comes with Bing to the fanciest paid models of Claude.

I have found them most common when asking a very specific question that hasn't be answered before,
like asking for a particular niche feature in CMake or Vulkan or Xcode.
Instead of answering "no it can't be done", I got the most probabilistic answer which suggested
I try to click a button that wasn't there or enable a feature flag that didn't exist.

The whole thing is akin to asking someone who's generally knowledgeable about a problem domain, but not the specific
software or library you are using. They would reply "yeah that sounds like a thing you should be able to do" because
it feels like a reasonable expectation to have and maybe is a feature in similar products. I suspect it's the same
reason why when writing code an LLM would try to call a non-existing API, because based on other languages it sounds
like C++ containers should have a `.sort()` function. Because it's the kind of thing you can do in Python and C#.
That's not the case because C++ makes a strong distinction between containers, iterators and algorithms,
but LLMs do not reason (despite their marketing calling it "reasoning") from first principles.

While this could be partially remedied by always asking for a primary source or citation, I dislike
the idea that one has to add magical incantations to their queries to get the right results.
It's a good laugh to make fun of "make no mistake" memes, until you start having to consider
similar things seriously. Plus models seem to get a new release every year or less, which
would probably require the user to revise all their rituals (or watch them become pointless
rituals that engineers do without remembering why).

Another issue I noted is that with connectors it's somewhat easy to get the LLM to start feeding
itself. For example when asked to find prior mentions of a recommendation I was writing for a client,
Claude was adamant that this was supported by past reports... until it turned out one of those
was the one I was actually writing. It was easy enough to catch because I had the breakdown with
sources, but had it just gave me numbers I could easily have created a self-reinforcing loop.
Likewise if coworkers were querying the same database and found my work-in-progress report, their
summaries may have taken it as gospel. Again, those robots are anything but intelligent, and often
you need to lay out some very basic things to avoid really stupid assumptions.

To finish on that topic, I also noticed a risk of telephone game happening with connectors that
plug into another LLM. Once I got Claude telling me there was concrete evidence that a certain
technical design was the result of a deliberate choice by the team, while it turned out it had
taken the summary of another AI at face value, and the real primary source was two users
speculating on why the module worked this way on the public forums. Again this is a tool
that is good at summarizing data, but not always good at selecting which data to trust.

## Coding?

Up until this paragraph my use cases have focused on research. But what about writing code?
After all, it is the next big thing™️ and it's coming any day now, isn't it? Simply put:
it's not very good.

While I have found LLMs useful for researching and planning code changes, my attempts at
actually making them write code have been quite lackluster. I found them to be slow and
expensive to generate, for a mediocre result.

In one use case, after a long discussion, I asked it to make an optimization refactor
where it would remove the `Update()` method of a `MonoBehaviour` derived object, put
those objects inside a `List<Foo>` owned by a manager class  and then run the equivalent
update in the manager class as a `for` loop. This is a fairly common optimization in Unity
games to avoid paying the cost of a (virtual) invocation of `MonoBehaviour.Update()` from
the C++ part of the engine to the managed C# script when you have many objects
of the same type.

Instead of doing just that, Claude created a `GameUpdateable` base class with an `OnUpdate()` method,
made `Foo` inherit from it, then stored the array as `List<GameUpdateable>` in the manager and made
a loop calling `GameUpdateable.OnUpdate()`. While this could still be devirtualized by some C# backends,
and while using a pure C# call is still a win compared to a C++ to C# call, it was still a more complex
and over-engineered solution than the one I had asked. Instead of "just doing the thing", Claude had decided
to apply some OOP design pattern that was uncalled for. And that wasn't a really hard thing to do, the
changes were localized to maybe 2 or 3 files.

Research tells me this is likely due to the fact that the training data for game development
(and to some extent all native programming) is poor. Languages that favour OOP
[dominate the training data](https://arxiv.org/html/2411.04905v1#A1e), even after applying some
filtering weights.

Worse, when it comes to games, there are barely any primary sources available at all.
The last AAA game to be open sourced was [Doom 3](https://github.com/id-Software/doom-3), a
title that released in 2004, 22 years ago. It doesn't even have multithreading<sup>[4](#myfootnote4)</sup>!
Other classics that have been released on Github over the years are usually from the 90s, complete
with software rasterization or fixed pipeline OpenGL 1.2 if you're lucky.

If you ask an LLM to write game code today, chances here it's been trained on hobby projects,
game jams and tutorial demos, assuming it's been trained on games at all.

Friends who work on custom engines with bespoke scripting languages told me that most
script they got out of LLMs was entirely at odds with the way they write it internally, because
the only available training data for their in-house scripting language is found in mods.

Oh, and when I mentioned expensive, remember than outputs use 5 to 10 times more tokens
than inputs (or maybe output tokens are priced 5-10x compared to input, which is the same).
So while writing summaries is not too token-hungry because it ingests much more
than it produces, asking it to write code is the opposite.

## Coding assistant

I was recently reading about the work of British management consultant Stafford Beer
<sup>[5](#myfootnote5)</sup>. His whole field of management cybernetics<sup>[6](#myfootnote6)</sup>
can be very roughly summarized by the idea that a manager (and at a larger scale, a company)
is limited in its ability to make good decisions by its capacity to process information
about its own workings, its customers and suppliers and any other factors that can influence the outcomes.
One important factor is that there is always more information (signal) coming in as the circumstances
change and so if something cannot be processed, it will likely be dropped, and if too much
is dropped (for example if you take the extreme case of reducing all inputs into a green or red signal),
bad things happen. You need enough variety in the data that is acted upon to avoid missing something crucial.

In a similar (but imperfect, I'll admit) analogy, dropping in a new codebase requires
processing a lot of data to make the right call. That happens each time you change jobs or projects,
and when you're in the consulting business that could be quite frequent. And so, when trying
to make an impact on a project in a short amount of time, one is often limited by how much
information they can absorb. In those environments, having an LLM that can help exploring
the code and pulling on some strings to see where they lead can be quite helpful. Maybe
we should have kept with the original "AI coding assistant" name instead of trying to make
them do everything...

I did use it to try and help me fix bugs, and again found it useful in areas where
I didn't know much since it helped my understanding of some code, but again I need
to stress the need to double check everything. I can usually spot nonsensical claims
when it comes to the things I've with before, but for new topics it's much more
dangerous.

To give some examples, I gave it a go on my personal rendering project and I got mixed results
with the bugs I was trying to understand.

The first was a culling bug due to mixed-up handedness. I was fairly easy to find if,
unlike me, you had not stubbornly refused to learned the first principles math behind it.
I'm sure someone who's been doing graphics for a few years would have caught it quickly.
When asked it to write a fix, it came up with a textbook solution that worked, but was much less efficient
than the project I had been taking inspiration from, despite Claude having parsed the source in its
context.

The second was a PBR lighting issue that made it suggest we rewrite the whole lighting, move
the main light position and add environment lighting, all to conclude after a few hours
that one of the assets had a faulty metallic/roughness texture and no amount of changing the
lighting code would fix it properly. Again, it's easy to be led on a wild goose chase by a
confident sounding LLM when you aren't yourself an expert on the topic.

I keep this quote in mind a lot lately:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">It&#39;s interesting how AI is constantly providing false information and incorrect statements about my area of expertise. Fortunately, it&#39;s very useful and always right about topics I know very little about.</p>&mdash; pikuma.com (@pikuma) <a href="https://x.com/pikuma/status/2068080467450937371?ref_src=twsrc%5Etfw">June 19, 2026</a></blockquote>

## Sustainability

AI companies are all [losing money](https://www.wheresyoured.at/the-openai-bubble/).
Of course they all have promises that they will eventually turn a profit and make an insane
return on investments, but so far they haven't. The tokens you get are making
a _marginal_ profit<sup>[7](#myfootnote7)</sup>, but overall they're running at a loss.
Given that hardware isn't getting cheaper and models still need to be trained all the time,
at the moment it doesn't look like the current prices are sustainable.

That's all to say that signs point towards the LLMs getting more expensive in the future,
not less. Like every tool you can buy or rent to help you on the job, there should be
a cost vs benefit calculation to be kept in mind, and so far it doesn't seem to be
happening outside of companies mandating everyone uses AI, followed by a counter order
a few months later when the
[bills come due](https://www.bloomberg.com/news/articles/2026-06-02/uber-caps-usage-of-ai-tools-like-claude-code-to-cut-costs).

In a past life I had to argue every year to renew a license for a profiling tool
that cost about 20 EUR a month. I've heard since that everyone at the company is now getting
a Claude subscription, even non-programmers. This is the kind of decision that baffles me.
Management will not trust its engineers to spend their money on the tools they say they need,
but then will tell everyone they have to use a shiny expensive toy they didn't request.

Is an LLM subscription more versatile than a profiler license for the average programmer?
Probably. I found it useful to speed up research in areas I wasn't familiar with. Would I
use it every day if I was working on the same project all the time? Probably not. Did it
radically change how I code? Not really. Is it going to replace programmers? I don't think
it's there, and at this point I'm seriously doubting it ever will be.

And if you find yourself using it a lot to write boilerplate code that you don't really care
to review, maybe you need to invest in a better API.

---

<a name="myfootnote1"><sup>1</sup></a>: I know that LLMs can also generate image and video formats. But it's 
out of scope for the purpose of this article. Also the results suck. AI "art" is a non-starter in my book.

<a name="myfootnote2"><sup>2</sup></a>: If you like cooking and sometimes search online for recipes,
you know exactly what I mean.

<a name="myfootnote3"><sup>3</sup></a>: Email on the other hand has seen some progress since the time where
you had to install Google Desktop to be able to find anything in Outlook, probably because the biggest email
providers today also run a search engine and an AI research division.

<a name="myfootnote4"><sup>4</sup></a>: There's some in the [BFG edition](https://github.com/id-Software/DOOM-3-BFG)
that was made in 2012, but it's more of a port than a new game.

<a name="myfootnote5"><sup>5</sup></a>: If you've ever heard the phrase "the purpose of a system is what it does",
that's him.

<a name="myfootnote6"><sup>6</sup></a>: Beer was influenced by W. Ross Ashby, who himself built upon
 the works of Claude Shannon. Yes. that's the Claude your LLM is named after. We've come full circle, in a way.

<a name="myfootnote7"><sup>7</sup></a>: Meaning they're profitable if only consider the direct costs of
running the model, which excludes a lot of R&D and hardware spending.