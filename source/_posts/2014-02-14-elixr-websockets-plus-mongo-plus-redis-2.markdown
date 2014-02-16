---
layout: post
title: "Elixir websockets+Mongo+Redis (2)"
date: 2014-02-14 13:04:46 +0000
comments: true
categories: [programming, elixir]
published: true
---

<br/>
### Getting Started

In my previous [post](http://pap.github.io/blog/2014/02/13/elixr-plus-mongo-plus-redis-plus-websockets/) i explained what drove me into an implementation of a small OTP app to subscribe json messages in a Redis channel and foward that json to one or several clients.

I'll walktrough some of the code that you can find [here](https://github.com/pap/ws_pub_sub).

The elixir app is under in [this folder](https://github.com/pap/ws_pub_sub/tree/master/ws_pub_sub).


I will assume you already know how to get the app dependencies and how to compile them to get a working elixir app.


The [webpage](https://github.com/pap/ws_pub_sub/tree/master/webpage) folder has a webpage with bullet.js to give you a jumpstart.

To start the webpage, if you have python installed, you can run `python -m SimpleHTTPServer` inside the webpage folder and you will get it at `http://localhost:8000`.

<br/>

### Bootstraping

As usual in a `mix` project you will have a `mix.exs` file and the application modules implemented under the `lib` dir.

<br/>
__mix.exs__

```elixir
def application do
  [
    mod: { WsPubSub, [] },
    applications: [
      :crypto,
      :compiler,
      :syntax_tools,
      :cowlib,
      :ranch,
      :cowboy,
      :bullet,
      :eredis,
      :bson,
      :mongodb,
      :exredis,
      :exjson,
      :jsex,
      :jsx
    ]
  ]
end
```

Lines 4 to 19:
Applications defined here are started with the ws_pub_sub app.

Originally i was starting these apps in the start/2 function in `lib/ws_pub_sub.ex` but this way feels more natural.

<br/>
__ws_pub_sub.ex__

```elixir
defmodule WsPubSub do
  use Application.Behaviour

  def start(_type, _args) do
    # initialize cowboy
    setup_cowboy
    # Initializes the ETS to store connected users
    _table = ConnectionTable.init
    WsPubSub.Supervisor.start_link
  end

  defp setup_cowboy do
    my_dispatch = :cowboy_router.compile([
                    {:_, [{"/websocket", :buller_handler, [{:handler, WsHandler}]}]}
                  ])
    # NOTE: to listen in port 80 you probably need to run the app as sudo
    # for demo purpose i'll start in port 8088
    {:ok, _} = :cowboy.start_http(
                 :http,
                 100,
                 [{:port, 8088}],
                 [{:env, [{:dispatch, my_dispatch}]}]
               )
  end
end
```

In line 6 we initialize cowboy registering a websocket handler
module `WsHandler` with bullet_handler behaviour.

In line 8 we create an ETS table to hold the data (PID + Session Key)
of all authenticated clients.
The `ConnectionTable.ex` module has a few functions defined to make the
use of ets tables more elixir like and abstract the call of erlang
functions.
It allows us to call `ConnectionTable.insert(key, value)`
instead of `:ets.insert(@table_id, {key, value})`
<br/>
<br/>
### Websockets

The websocket handler is implemented in `lib/ws_pub_sub/ws_handler.ex`
Authorizing via mongo could definetly be extracted to its own module but
i opted to keep the mongo lookup here to allow easier browsing.

<br/>
__ws_handler.ex__

```elixir
def init(_transport, req, _opts, _active) do
  _ = :erlang.send_after(@period, self, :refresh)
  {qs, req2} = :cowboy_req.qs(req)
  # Read the querystring from request
  state = State[guid: qs]
  # check if key exists
  connect(qs)
  {:ok, req2, state}
end

defp connect(key) do
  case mongo_auth(key) do
    :ok ->
      # Key exists. User is allowed to receive updates.
      ConnectionTable.insert(key, :erlang.pid_to_list(self))
      # Publish in Redis:
      # You can notify on a given channel that a user was added
      # to the connected users table
      :global.whereis_name(:pubsub_exredis_client)
        |> Exredis.Api.publish "#{@redis_ws_in_chn}", "#{key}"
    :not_found ->
      #TODO: notify user, log attempt etc...
  end
end
```

Line 3:
We get the querystring from `cowboy_req`.

Line 7:
We pass the querystring to mongo to chek if the given key exists.

In `lib/ws_pub_sub/ws_handler.ex` you will find the mongo connection and query code under
`mongo_auth(key)`.
I think it's easy to follow so i'll skip it ... if you have any doubt open an
issue on the github repo.

If the key is found we add the user key an pid to the `connection table`
and in line 19 we publish a message in a given redis channel to notify
that a new user is added to the connected users.

That takes us into the `RedisPubSub` module.

This module is implemented with gen_server behaviour and it is added to
the app supervisor. This way we can recover from any unexpected failure,
and keep on listening to messages !

<br/>
__redis_pub_sub.ex__

```elixir
def init(_options\\[]) do
  client_sub = Exredis.Sub.start(@redis_host, @redis_port)
  client = Exredis.start(@redis_host, @redis_port)
  # Register the PubSub client so it is accessible to publish connection
  # and disconnection notifications
  :global.register_name(@name, client)
  _pid = Kernel.self
  state = State[client: client, client_sub: client_sub]
  NOTE: send() is defined in another module (MsgPusher)
  # the function is "registered" as the handler for any message arriving to
  # redis @notification_channel
  client_sub |>
    Exredis.Sub.subscribe "#{@notification_channel}", fn(msg) -> MsgPusher.send(msg) end
  {:ok, state}
end
```

In this particular file i would like to point out lines 12 and 13.

We subscribe a redis channel and the function we pass as the one to be
executed everytime a message arrives `fn(msg) -> MsgPusher.send(msg) end`
simply invokes the send/1 function defined under `MsgPusher` module.

If you want to see some JSON getting into your browser open a redis console (`redis-cli`).

Inside redis-cli enter:

```
PUBLISH "my_channel" "{\"recipients\":[\"00721b2a-046c-4ecc-a5df-5f808cc6c58f\"],\"data\":{\"entry\":{\"id\xe2\x80\x9d:\xe2\x80\x9d123\xe2\x80\x9d,\xe2\x80\x9dcomments\":0,\"tags\":0}}}"`
```

As the included example webpage indicates the websocket connection will only be established(authorized) if you have an entry in MongoDB with `_id:
00721b2a-046c-4ecc-a5df-5f808cc6c58f`

The default database is `myDB` and the collection `myCollection`.

