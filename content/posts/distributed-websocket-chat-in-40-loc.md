---
title: "Distributed websocket chat in 40 lines of code"
date: 2019-02-27T20:15:25+03:00
author: Vlad Faust
tags: [Crystal, Onyx Framework]
---

In this article you will find out how to easily create a distributed websocket chat application with [Crystal](https://crystal-lang.org) and [Onyx Framework](https://onyxframework.org).

<!--more-->

Crystal has already proven its superiority regarding to websockets performance in Serdar Doğruyol's [blog post](http://serdardogruyol.com/benchmarking-and-scaling-websockets-handling-60000-concurrent-connections-with-kemal) -- it is able to handle up to 60000 concurrent connections on a 2 GB DigitalOcean droplet.

It works great in a single process, but what if you want to scale your application? It is very simple with [Onyx::REST](https://github.com/onyxframework/rest) [`Channel`](https://api.onyxframework.org/rest/Onyx/REST/Channel.html) and [Onyx::EDA](https://github.com/onyxframework/eda). This is a complete code of the application:

```crystal
# src/onyx-chat.cr
#

require "onyx/rest"
require "onyx/eda"

Onyx.channel(:redis) # You'll need REDIS_URL variable to be set

struct Message
  include Onyx::EDA::Event

  getter username, content

  def initialize(@username : String, @content : String)
  end
end

class Chat
  include Onyx::REST::Channel

  params do
    query do
      type name : String
    end
  end

  def on_open
    Onyx.subscribe(self, Message) do |message|
      socket.send("#{message.username}: #{message.content}")
    end
  end

  def on_message(message)
    Onyx.emit(Message.new(params.query.name, message))
  end

  def on_close
    Onyx.unsubscribe(self)
  end
end

Onyx.ws "/", Chat
Onyx.listen
```

First terminal:

```sh
> env PORT=5000 REDIS_URL=redis://localhost:6379 crystal src/onyx-chat.cr
 INFO [21:25:38.483] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
```

Second terminal (note another port):

```sh
> env PORT=5001 REDIS_URL=redis://localhost:6379 crystal src/onyx-chat.cr
 INFO [21:25:38.483] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5001
```

Third terminal:

```sh
> wscat --connect ws://localhost:5000?name=Alice
connected (press CTRL+C to quit)
> Hello!
< Alice: Hello!
< Bob: Hi!
>
```

Fourth terminal:

```sh
> wscat --connect ws://localhost:5001?name=Bob
connected (press CTRL+C to quit)
< Alice: Hello!
> Hi!
< Bob: Hi!
>
```

These are two separate websocket chat processes which use Redis as a back-end for synchronisation, just in 40 lines!

Note that you'll need Redis >= 5 to make Onyx::EDA work with it.

I use [wscat](https://www.npmjs.com/package/wscat) to test the websockets.

Crystal dependencies you'd need:

```yaml
# shard.yml
#

dependencies:
  onyx:
    github: onyxframework/onyx
    version: ~> 0.1.0
  onyx-rest:
    github: onyxframework/rest
    version: ~> 0.6.0
  onyx-eda:
    github: onyxframework/eda
    version: ~> 0.2.0
```

That's all for today!
