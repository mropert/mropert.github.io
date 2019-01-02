---
layout: post
title: Gamedev introduction to "Modern" C++
tags: [cpp]
description: >
  Is the C++ committee out of touch with its users, or is the game development crowd out of touch
  with the rest of the C++ community? Following the recent debate on social media, I offer a small
  introduction of that "Modern" C++ thingie.
author: mropert
---

I was initially going to write a response to Aras Pranckevičius' 
[Modern C++ Lamentations](http://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/) but the great
Ben Dean [beat me to it](http://www.elbeno.com/blog/?p=1598).

Reading the whole discussion on social media, I felt like an UFO, as I am part of the videogame
industry but don't believe that C++ is headed in the wrong direction or that the committee as lost
touch with the community. And I think it is probably a matter of background.

I didn't start my career in videogames. My engineering thesis was on operating systems design,
then I worked for the web (in C) and then in finance (in C then C++), in total more than 10 years
before I joined a videogame company.
Before opening the codebase of a videogame, I went to a couple big C++ conferences, spoke at some of them,
organized meetups for my local user group and of course in the process met a lot of people who are committee
members, voters, writers or simply influencers.

As Ben put it, I feel like my fellow gamedevs could gain a lot by participating more in the C++
community that's out there, and in turn bring out topics that may have been overlooked.

When writing my original thoughts, I also realized a lot of discussion reminded me of Dan Saks'
[Talking to C Programmers about C++](https://www.youtube.com/watch?v=D7Sd8A6_fYU), so before
I suggest any other resource, I want to recommend this one. It explains why debugging isn't the #1
priority for the committee and explains better than I could why discussing technical points with
different perspectives can be difficult.

## The Standard and the Committee

C++ is an international standard. As such, it falls under the rulings of ISO and is one of the few
programming languages to do so (C being the over one). This comes with a few organizational
constraints.

While the final draft has to be voted by people representing national standard organizations
(ANSI, AFNOR, BFI...), all the editing process is open to anyone who wants to participate.
Participants usually include national bodies representatives, but also academics and engineers
from all across the world.

So how does one participate? Well that's easy, they just have to show up. The C++ committee usually
meets thrice a year, most often in the US or in Europe and anyone who comes can participate in the
meetings and debates. The only thing they can't do is vote in the formal polls since ISO requires
to be a member or representative of a national organization.

Most participants are sponsored by their company or university to attend, but some do come
at their own expenses. There are also a few non-profit organizations that sponsor people to attend,
such as the [C++ Alliance](https://cppalliance.org/).

Being unable to find the time or sponsorship to attend is absolutely not a showstopper. In fact,
a lot of work in done between meetings, by email and other electronic exchanges. Meetings are
usually about discussing and voting proposals, while all the writing and editing is done outside.
One can perfectly submit a paper, address comments, and then find a champion in the attendees
to defend it at committee meetings. More can be found [here](https://isocpp.org/std/submit-a-proposal).

Finally, since the C++ standard is a big piece, work is usually split by topic. WG21
has created a number of study groups over the years to discuss particular bits of the language.
The one that should interest my readers the most is SG14: Game Development and Low Latency.
It's free to join, all the discussions are available on the
[mailing-list](https://groups.google.com/a/isocpp.org/forum/?fromgroups=#!forum/sg14)
and there's a teleconference most Wednesdays on European evening time.

In short, most of the process that leads up to the adoption of a new C++ standard is open to
anyone who wants to follow or participate. For more information, refer to the
[ISO C++ website](https://isocpp.org/std).

## The community

Not everyone in the C++ community participates in the standard, nor do they have to. Getting in
touch with fellow developers is already a big plus.

In my opinion, the best way is to attend conferences. Here's a incomplete schedule for 2019:
* [CppOnSea](https://cpponsea.uk/) (Folkestone, UK), February 4-6th
* [ACCU](https://conference.accu.org/) (Bristol, UK), April 10-13th
* [C++Now](http://cppnow.org/) (Aspen, CO, USA), May 5-10th
* [CppCon](https://cppcon.org/) (Aurora, CO, USA), September 15-20th
* [Pacific++](https://pacificplusplus.com/) (Australia/New Zealand), usually mid October
* [Meeting C++](https://meetingcpp.com/) (Berlin, Germany), usually mid November

Most employers would gain a lot by sending their developers there, but if that's not an option,
the conferences usually offer some student and diversity tickets at a discount. Of course
the best way is to
[submit a talk](https://playfulprogramming.blogspot.com/2018/11/how-to-speak-at-conference.html)
and get most/all expenses paid by the conference!

Outside of those special gathering, there's probably a
[C++ Meetup happening nearby](https://www.batchgeo.com/map/dbeba81134e05db15288b6ba66f30e0a).
And if not, I would recommend starting one. I know personally a couple people who launched one
because they couldn't find it, and quickly got dozens of attendees.

Finally, there's the [Cpplang Slack](https://cpplang.now.sh/)!

## Online resources

Should you miss a conference, or simply be curious of what happened there, most (if not all) of
them publish all the recordings freely on youtube:
* [ACCU](https://www.youtube.com/channel/UCJhay24LTpO1s4bIZxuIqKw/videos)
* [CppCon](https://www.youtube.com/user/CppCon/videos)
* [C++Now](https://www.youtube.com/user/BoostCon/videos)
* [MeetingCpp](https://www.youtube.com/user/MeetingCPP/videos)
* [Pacific++](https://www.youtube.com/channel/UCrRR5mU5aqvtZAuEGYfdTjw/videos)

For readers who commute to work, or go running, or whatever else, it's always good to save up a
couple of [CppCast](http://cppcast.com/) episodes to listen to. Each week Rob and Jason interview
someone from the community and share recent news about C++.

Also, since historically the gamedev community doesn't trust its compiler, the
[Goldbolt](https://godbolt.org/)<sup>1</sup> is an invaluable resource.

And of course, you should always measure, so better run the code through [QuickBench](http://quick-bench.com/).

## Mythbusting

In closing, I'd like to suggest a couple talks that give some perspective on where the C++
community is heading:
* [Using C++17 to make a video game on a legacy platform](https://www.youtube.com/watch?v=zBkNBP00wJE) by Jason Turner
* [How smart can compilers be?](https://www.youtube.com/watch?v=bSkpMdDe4g4) by Matt Godbolt
* [Ranges for the Standard Library](https://www.youtube.com/watch?v=mFUXNMfaciE) by Eric Niebler
* [Writing good C++ by default](https://www.youtube.com/watch?v=hEx5DNLWGgA) by Herb Sutter

Happy watching and welcome here, I hope to see you around soon!

<sup>1</sup> Sorry Matt, I know it's called Compiler Explorer but it just doesn't stick :(