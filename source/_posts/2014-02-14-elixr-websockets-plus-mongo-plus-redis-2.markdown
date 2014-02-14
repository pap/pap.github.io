---
layout: post
title: "Elixr websockets+Mongo+Redis (2)"
date: 2014-02-14 13:04:46 +0000
comments: true
categories: [programming, elixir]
published: false
---

##Getting Started

<br/>
In my previous [post](http://pap.github.io/blog/2014/02/13/elixr-plus-mongo-plus-redis-plus-websockets/) i explained what drove me into an implementation of a small OTP app to subscribe json messages in a Redis channel and foward that json to one or several clients.

I'll walktrough some of the code that you can find [here](https://github.com/pap/ws_pub_sub).

The elixir app is under in [this folder](https://github.com/pap/ws_pub_sub/tree/master/ws_pub_sub).


I will assume you already know how to get the app dependencies and how to compile them to get a working elixir app.


The [webpage](https://github.com/pap/ws_pub_sub/tree/master/webpage) folder has a webpage with bullet.js to give you a jumpstart.

To start the webpage, if you have python installed, you can run `python -m SimpleHTTPServer` inside the webpage folder and you will get it at `http://localhost:8000`.

<br/>
##Bootstraping

<br/>
As usual in a `mix` project you will have a `mix.exs` file and the application modules implemented under the `lib` dir.

