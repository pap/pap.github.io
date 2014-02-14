---
layout: post
title: "Elixir+Mongo+Redis+Websockets"
date: 2014-02-13 23:20:23 +0000
comments: true
categories:
---

A few months ago a fellow developer was architecting a new app (soon to be launched) and he had a problem to solve:

  __Provide realtime updates in one or several devices after an event triggered by one of those devices__

It had to be fault tolerant, performant... well, business as usual !

By the time i was quite interested in Erlang/Elixir, and i offered to prototype a solution. It just felt the right tool for the job.

All in all it was not that different of the canonical "hello world" of the Erlang VM languages ... the chatroom example.

After a few days trying to find documentation and examples on websocket use i was able to prototype something in Erlang with [cowboy](https://github.com/extend/cowboy).

Then things changed a bit. Prior to websocket connection "validation" a user should have a valid key stored in a database.
And there would be a Redis channel to publish messages that had to be routed to a subset of the registered connected users.

It was time to refactor.

I opted to use [Elixir](http://elixir-lang.org) and i've also added [bullet](https://github.com/extend/bullet) [^1] to my previous cowboy setup.
[^1]:_"Bullet is a Cowboy handler and associated Javascript library for maintaining a persistent connection between a client and a server."_
I also had to integrate a few libraries to:

  * parse and output json
  * connect to a database to verify identity
  * subscribe to a Redis channel and listen to messages

I knew it would be a risk to chose a language which being actively developed (currently in version 0.12.4) but it just felt right !

On my next post i'll dive into implementation details.
