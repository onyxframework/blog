---
title: "Creating JSON APIs with Onyx. Part 1 — The First Endpoint"
date: 2019-02-23T16:21:06+03:00
lastmod: 2019-03-11
description: In this series of tutorials, we'll show you how to build a simple JSON API for a Todo items list with Crystal programming language and Onyx Framework.
author: Vlad Faust
tags: [Crystal, Onyx Framework]
---

In this series of tutorials, we will show you how to build a simple JSON API for a Todo items list with [Crystal programming language](https://crystal-lang.org/) and [Onyx Framework](https://onyxframework.org/).

<!--more-->

## Tutorial contents

1. Part 1 — The First Endpoint (*this article*)
2. [Part 2 — CRUD](/posts/creating-json-apis-with-onyx-part-2)

---

💡 Haven't heard of Crystal yet or not sure whether to use it? You should check the recent [Why Crystal](/posts/why-crystal/) blogpost then.

💡 If you stuck, then ask your question in the [Gitter room](https://gitter.im/onyxframework) or on [Twitter](https://twitter.com/onyxframework)!

---

**Table of contents:**

1. [Installing Crystal](#installing-crystal)
2. [Hello, world!](#hello-world)
3. [Initializing an application](#initializing-an-application)
4. [Adding dependencies](#adding-dependencies)
5. [The first endpoint](#the-first-endpoint)
6. [Creating an Action](#creating-an-action)
7. [Adding a JSON view](#adding-a-json-view)
8. [Adding query params](#adding-query-params)

### Installing Crystal

Crystal must be installed on your computer to run Crystal programs, just like with the most of the other programming languages.

The official documentation has a thorough guide on how to install Crystal -- <https://crystal-lang.org/reference/installation/>, please follow it, check the language is istalled with `crystal -v` and move to the next step.

### Hello, world!

To run a simple Crystal program, create a file named `hello_world.cr` and put this line within it:

```crystal
puts "Hello, world!"
```

Looks familiar, doesn't it? Now run the program:

```plaintext
> crystal hello_world.cr
Hello, world!
```

Congratulations, you've successfully created and run your first Crystal program!

#### ICR

You may wonder if Crystal has an alternative to the [IRB](https://en.wikipedia.org/wiki/Interactive_Ruby_Shell). Well, yes, with some implications. Check out [ICR](https://github.com/crystal-community/icr):

```sh
icr(0.27.2) > puts "Hello, world!"
Hello, world!
```

### Initializing an application

New applications are usually initiated with `crystal init app appname`. Let's create a new application called "todo-onyx":

```sh
> crystal init app todo-onyx && cd todo-onyx
```

You can check that everything is OK with this command (it should return nothing):

```sh
> crystal src/todo-onyx.cr
```

For the sake of the tutorial, rename the `todo-onyx.cr` file to `server.cr` and put this line within (for now):

```crystal
puts "I'm the server"
```

```sh
> crystal src/server.cr
I'm the server
```

The directory should look like this:

```plaintext
.
├── .editorconfig
├── .gitignore
├── LICENSE
├── README.md
├── shard.lock
├── shard.yml
├── spec
│   ├── spec_helper.cr
│   └── todo-onyx_spec.cr
├── src
│   └── server.cr
└── .travis.yml
```

### Adding dependencies

It's time to add some ~~gems~~ shards! As we're using [Onyx Framework](https://onyxframework.org), modify your `shard.yml` file adding `dependencies` section like this:

{{< highlight yaml "hl_lines=9-15" >}}
targets:
  todo-onyx:
    main: src/todo-onyx.cr

crystal: 0.27.2

license: MIT

dependencies:
  onyx:
    github: onyxframework/onyx
    version: ~> 0.3.0
  onyx-http:
    github: onyxframework/http
    version: ~> 0.7.0
{{< / highlight >}}

Run `shards install` afterwards:

```sh
> shards install
```

To learn more about Crystal shards, see [the docs](https://github.com/crystal-lang/shards).

Now as you have all the needed dependencies, let's write some real code.

### The first endpoint

Let's make the server respond to the `GET /` request. Replace the contents of `src/server.cr` with this code:

```crystal
require "onyx/http"

Onyx.get "/" do |env|
  env.response << "Hello, Onyx!"
end

Onyx.listen
```

Now run the server in a separate terminal:

```sh
> crystal src/server.cr
 INFO [19:40:43.668] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
```

And check the endpoint:

```sh
> curl http://127.0.0.1:5000
Hello, Onyx!
```

Marvelous! Now stop the server with Ctrl+C command:

```sh
> crystal src/server.cr
 INFO [19:40:43.668] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
 INFO [19:41:19.513] [c6ff5cc0]    GET / 200 90μs
^C
 INFO [19:42:11.264] ⬛ Onyx::HTTP::Server is shutting down!
```

You should restart the server **manually** every time you make a change.

### Creating an Action

In [Onyx::HTTP](https://onyxframework.org/http), you can encapsulate endpoints into separate objects. They usually isolate business logic from the rendering layer.

Create a new file at `src/endpoints/hello.cr` and put the following code into it:

{{< highlight plaintext "hl_lines=2-3" >}}
└── src
    ├── endpoints
    │   └── hello.cr
    └── server.cr
{{< / highlight >}}

```crystal
struct Endpoints::Hello
  include Onyx::HTTP::Endpoint

  def call
    context.response << "Hello, Onyx!"
  end
end
```

Modify the `src/server.cr` file:

{{< highlight crystal "hl_lines=2 4" >}}
require "onyx/http"
require "./endpoints/**"

Onyx.get "/", Endpoints::Hello

Onyx.listen
{{< / highlight >}}

Should you check it with curl, the response will be the same, but now we have a endpoint defined as a separate object.

```sh
> curl http://127.0.0.1:5000
Hello, Onyx!
```

### Adding a JSON view

Currently our action actually does some rendering. To separate it into another layer, we'll make use of the [Onyx::HTTP views](https://docs.onyxframework.org/http/views) concept.

Views usually don't know anything about the application logic, all they do is rendering their payload.

Create a new file at `src/views/hello.cr`:

{{< highlight plaintext "hl_lines=5-6" >}}
└── src
    ├── endpoints
    │   └── hello.cr
    ├── server.cr
    └── views
        └── hello.cr
{{< / highlight >}}

```crystal
struct Views::Hello
  include Onyx::HTTP::View

  def initialize(@who : String)
  end

  json message: "Hello, #{@who}!"
end
```

Now we can use this view in our action:

{{< highlight crystal "hl_lines=5" >}}
struct Endpoints::Hello
  include Onyx::HTTP::Endpoint

  def call
    return Views::Hello.new("Onyx")
  end
end
{{< / highlight >}}

{{< highlight crystal "hl_lines=3" >}}
require "onyx/http"

require "./views/**"
require "./endpoints/**"

Onyx.get "/", Endpoints::Hello

Onyx.listen
{{< / highlight >}}

The response should now return a JSON string:

```sh
> curl http://127.0.0.1:5000
{"message":"Hello, Onyx!"}
```

### Adding query params

`Onyx::HTTP::Endpoint` module has a powerful `params` DSL which allows to define strongly-typed parameters. Currently the endpoint always returns `"Hello, Onyx!"`. Let's change it with a dynamic query parameter:

{{< highlight crystal "hl_lines=4-8 11-12" >}}
struct Endpoints::Hello
  include Onyx::HTTP::Endpoint

  params do
    query do
      type who : String = "Onyx" # The default value is "Onyx"
    end
  end

  def call
    # At this point, `params.query.who` is guaranteed to be a String
    return Views::Hello.new(params.query.who)
  end
end
{{< / highlight >}}

From now on, the URL query affects the returning value:

```sh
> curl http://127.0.0.1:5000
{"message":"Hello, Onyx!"}
> curl http://127.0.0.1:5000?who=Crystal
{"message":"Hello, Crystal!"}
```

That's all for part I of this tutorial. You've learned how to create REST API endpoints with separate business and rendering layers. The complete source code for this (and other) part is available at [GitHub](https://github.com/vladfaust/onyx-todo-json-api/tree/part-1).

[Continue to part 2 →](/posts/creating-json-apis-with-onyx-part-2)
