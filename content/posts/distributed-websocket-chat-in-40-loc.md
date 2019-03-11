---
title: "Distributed websocket chat in 40 lines of code"
date: 2019-02-27T20:15:25+03:00
lastmod: 2019-03-11
author: Vlad Faust
tags: [Crystal, Onyx Framework]
---

In this article you will find out how to easily create a distributed websocket chat application with [Crystal](https://crystal-lang.org) and [Onyx Framework](https://onyxframework.org).

<!--more-->

[Crystal](https://crystal-lang.org) is a rapidly growing compiled language with speed of C and Ruby-inspired syntax. It has already proven its superiority regarding to websockets performance in Serdar Doğruyol's [blog post](http://serdardogruyol.com/benchmarking-and-scaling-websockets-handling-60000-concurrent-connections-with-kemal) -- it is able to handle up to 60000 concurrent connections on a 2 GB DigitalOcean droplet:

<div class="twitter-widget-wrapper">
{{< tweet 797835943864573952 >}}
</div>

It works great in a single process, but what if you wanted to scale your application? It is very simple to do so with [Onyx::HTTP](https://onyxframework.org/http) channels and [Onyx::EDA](https://onyxframework.org/eda) events.

Note that Onyx::EDA relies on Redis [Streams](https://redis.io/topics/streams-intro) feature, that's why it requires Redis version **>=5**. I use [wscat](https://www.npmjs.com/package/wscat) to test the websockets in the terminal in this article.

> The repository is available at [github.com/vladfaust/onyx-40-loc-distributed-chat](https://github.com/vladfaust/onyx-40-loc-distributed-chat).

Here goes the complete code of the server:

```crystal
require "onyx/http"
require "onyx/eda"

Onyx.channel(:redis) # You'll need REDIS_URL variable to be set

struct Message
  include Onyx::EDA::Event

  getter username, content

  def initialize(@username : String, @content : String)
  end
end

class Chat
  include Onyx::HTTP::Channel

  params do
    query do
      type username : String
    end
  end

  def on_open
    Onyx.subscribe(self, Message) do |message|
      socket.send("#{message.username}: #{message.content}")
    end
  end

  def on_message(message)
    Onyx.emit(Message.new(params.query.username, message))
  end

  def on_close
    Onyx.unsubscribe(self)
  end
end

Onyx.ws "/", Chat
Onyx.listen(port: ENV["PORT"].to_i) # You'll also need PORT variable
```

First terminal:

```sh
> env PORT=5000 crystal src/onyx-chat.cr
 INFO [21:25:38.483] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
```

Second terminal (note another port):

```sh
> env PORT=5001 crystal src/onyx-chat.cr
 INFO [21:25:38.483] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5001
```

Third terminal:

```sh
> wscat --connect ws://localhost:5000?username=Alice
connected (press CTRL+C to quit)
> Hello! # Message sent from this terminal
< Alice: Hello!
< Bob: Hi!
>
```

Fourth terminal:

```sh
> wscat --connect ws://localhost:5001?username=Bob
connected (press CTRL+C to quit)
< Alice: Hello!
> Hi! # Message sent from this terminal
< Bob: Hi!
>
```

These are two separate websocket chat processes which use Redis as a back-end for synchronisation, in just in **40 lines of code**!

Crystal dependencies ([*shards*](https://github.com/crystal-lang/shards)) you'd need:

```yaml
dependencies:
  onyx:
    github: onyxframework/onyx
    version: ~> 0.3.0
  onyx-http:
    github: onyxframework/http
    version: ~> 0.7.0
  onyx-eda:
    github: onyxframework/eda
    version: ~> 0.2.0
```

If you like this article but have no experience in Crystal, then follow out [series of tutorials](/posts/creating-json-apis-with-onyx-part-1/) which would lead you through the basics.
