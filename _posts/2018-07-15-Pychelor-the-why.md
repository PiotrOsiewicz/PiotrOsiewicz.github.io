---
layout: post
title:  "Pychelor - the ,,Why''"
date:   2018-07-15 22:00:00 +0000
tags: [ Pychelor, Pychelor specification, projects]
---

*After some exposure to distributed systems, I have found a perfect subject for
my bachelors that involves one of my favourite subjects, statistics. This is
Pychelor in a nutshell - data analysis and distributed systems (yes, data
analysis is not equal to statistics).*

Pychelor's sole purpose is to make tracing events in distributed applications
easier, even in dire times of unmet deadlines and ringing phones.
It shall allow developers to trace the performance of their code in context thanks to
system load tracking. 

Let's use a simple distributed system (in a supervisor-worker relation), where
each participant of such system might have it's own log, either condensed into one file
(which I would call event log - even though it fits the following usage just as well)
or split into multiple logs per functionality.
Each worker is connected to one and only one supervisor, however one supervisor
can be connected to many workers.
Let us assume that should the communication between separate working units be
required, it will be handled by supervisor - a proxy.
Debugging such architecture is tricky for a variety of reasons:
* Ease of using logs for debugging purposes (when looking at one worker
  and one supervisor) is strictly dependent upon the
  implementation of supervisor's logger - should it condense multiple "event
  streams" (that is - multiple sources of events) into one file, searching through
  (quite often) few gigabytes of data (most of which is of no use, because it
  is not related to this particular worker) might be quite hard.

* On the other hand, splitting the data into multiple files yields no context
  on the events - it is possible to deduce simple bugs that are directly connected 
  with previous events in the log, however if the state of whole system comes into
  play, we are left empty-handed (unless you are willing to dig through five other
  files - but then.. are you lazy enough to call yourself a programmer?).

* Raw size of data (regardless of used logging model) and the method of adding
  new information to log file (which is - most of the time - simply appending
  the information to the end of file) makes old data easier to access than new
  data. Make no mistake - this is **related** to the first point, however \- 
  while both of these are concerned with the size of a log - the previous argument
  was closely connected with information noise.

* Logging directly from the process being observed means that - in case of
  failure - the log might get corrupted or lost altogether. Sharing single
  logger between multiple units (at whichever level of granularity)
  adds insult to injury in form of locking, which means that the event might not
  even be noticed. Surely, making robust software is hard.. The best we can do is try.

* Reading through logs can be just as cumbersome - sometimes it's hard to find
  relevant information in the noise of thread IDs, timestamps and all this good
  stuff. Hey, I'm *not* saying these are bad. That's part of logging! However, they
  are (in my opnion) programmer's best friend when enabled at just right moment \-
  which is after locating the source of the trouble.
  
Surprise, surprise - I cannot deal with any of these. Would you expect *that*?

However, as I have said previously, the best I can do is try. Which is why I am
gonna work on Pychelor. What I believe is key to solving aforementioned 
issues is to create a tool & bindings that would be:

* Distributed, but with rich context - logs should be bound at lowest level of
  granulation, thus making assembling relevant logs a breeze.

* Easy to mold - output logs should be prone to mutation at user's request.

* Robust - let's avoid system failure, shall we?

* Considerate of it's words - the last thing I'd like to see tommorow in my job is that
  logs took up half a terabyte, most of which is probably noise.. =)

* Lightweight - if possible, the impact of logging on the observed program
  itself should be negligible.

Someone has (probably) already came up with a better idea than mine, but hey.
They got it all wrong..

<div style="text-align:center">
<img align="middle" alt="xkcd-standards" src="https://imgs.xkcd.com/comics/standards.png"/>
<br/>
<i>As always, there is relevant xkcd </i>
</div>

Some of these things probably cancel out, but hey - I'm young, stupid and eager
to experiment! Let the rockoff begin!

