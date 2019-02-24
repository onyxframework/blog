---
title: "Creating JSON APIs with Onyx. Part 1 — The First Endpoint"
date: 2019-02-23T16:21:06+03:00
description: In this series of tutorials, we'll show you how to build a simple JSON API for a Todo items list with Crystal programming language and Onyx Framework.
author: Vlad Faust
tags: [Crystal, Onyx Framework]
---

In this series of tutorials, we'll show you how to build a simple JSON API for a Todo items list with [Crystal programming language](https://crystal-lang.org/) and [Onyx Framework](https://onyxframework.org/).

<!--more-->

## Tutorial contents

1. Part 1 — The First Endpoint (*this article*)
2. [Part 2 — CRUD](/posts/creating-json-apis-with-onyx-part-2)

---

:bulb: Haven't heard of Crystal yet or not sure whether to use it? You should check the recent [Why Crystal](/posts/why-crystal/) blogpost then.

:bulb: If you stuck, then ask your question in the [Gitter room](https://gitter.im/onyxframework) or on [Twitter](https://twitter.com/onyxframework)!

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

```plaintext
icr(0.27.2) > puts "Hello, world!"
Hello, world!
```

### Initializing an application

New applications are usually initiated with `crystal init app appname`. Let's create a new application called "todo-onyx":

```plaintext
> crystal init app todo-onyx && cd todo-onyx
    create  todo-onyx/.gitignore
    create  todo-onyx/.editorconfig
    create  todo-onyx/LICENSE
    create  todo-onyx/README.md
    create  todo-onyx/.travis.yml
    create  todo-onyx/shard.yml
    create  todo-onyx/src/todo-onyx.cr
    create  todo-onyx/spec/spec_helper.cr
    create  todo-onyx/spec/todo-onyx_spec.cr
Initialized empty Git repository in /home/todo-onyx/.git/
```

You can check that everything is OK with this command (it should return nothing):

```plaintext
> crystal src/todo-onyx.cr
```

For the sake of the tutorial, rename the `todo-onyx.cr` file to `server.cr` and put this line within (for now):

```crystal
puts "I'm the server"
```

```plaintext
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
│   ├── server.cr
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
    version: ~> 0.1.2
  onyx-rest:
    github: onyxframework/rest
    version: ~> 0.6.3
{{< / highlight >}}

Run `shards install` afterwards:

```plaintext
> shards install
Fetching https://github.com/onyxframework/onyx.git
Fetching https://github.com/gdotdesign/cr-dotenv.git
...
Fetching https://github.com/onyxframework/sql.git
Fetching https://github.com/crystal-lang/crystal-db.git
Installing onyx (0.1.2)
Installing dotenv (0.1.0)
...
Installing onyx-sql (0.6.2)
Installing db (0.5.1)
```

Now as you have all the needed dependencies, let's write some real code.

### The first endpoint

Let's make the server respond to the `GET /` request. Replace the contents of `src/server.cr` with this code:

```crystal
require "onyx/rest"

Onyx.get "/" do |env|
  env.response << "Hello, Onyx!"
end

Onyx.listen
```

Now run the server in a separate terminal:

```plaintext
> crystal src/server.cr
 INFO [19:40:43.668 #21269] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
```

And check the endpoint:

```plaintext
> curl http://localhost:5000
Hello, Onyx!
```

Marvelous! Now stop the server with Ctrl+C command:

```plaintext
> crystal src/server.cr
 INFO [19:40:43.668 #21269] ⬛ Onyx::HTTP::Server is listening at http://127.0.0.1:5000
 INFO [19:41:19.513 #21269] [c6ff5cc0]    GET / 200 90μs
^C
 INFO [19:42:11.264 #21269] ⬛ Onyx::HTTP::Server is shutting down!
```

You should restart the server manually every time you make a change.

### Creating an Action

In [Onyx::REST](https://github.com/onyxframework/rest), you can define separate endpoints called [`Action`](https://api.onyxframework.org/rest/Onyx/REST/Action.html)s. They usually isolate business logic from the rendering layer.

Create a new file at `src/actions/hello.cr` and put the following code into it:

{{< highlight plaintext "hl_lines=2-3" >}}
└── src
    ├── actions
    │   └── hello.cr
    └── server.cr
{{< / highlight >}}

```crystal
struct Actions::Hello
  include Onyx::REST::Action

  def call
    context.response << "Hello, Onyx!"
  end
end
```

Modify the `src/server.cr` file:

{{< highlight crystal "hl_lines=2 4" >}}
require "onyx/rest"
require "./actions/**"

Onyx.get "/", Actions::Hello

Onyx.listen
{{< / highlight >}}

Should you check it with curl, the response will be the same, but now we have a endpoint defined as a separate object.

```sh
> curl http://localhost:5000
Hello, Onyx!
```

### Adding a JSON view

Currently our action actually does some rendering. To separate it into another layer, we'll make use of the [Onyx::REST::View](https://api.onyxframework.org/rest/Onyx/REST/View.html) concept.

Views usually don't know anything about the application logic, all they do is rendering their payload. And the rendering mechanism depends on which renderer the server currently uses. It can be, for example, a JSON renderer.

Create a new file at `src/views/hello.cr`:

{{< highlight plaintext "hl_lines=5-6" >}}
└── src
    ├── actions
    │   └── hello.cr
    ├── server.cr
    └── views
        └── hello.cr
{{< / highlight >}}

```crystal
struct Views::Hello
  include Onyx::REST::View

  def initialize(@who : String)
  end

  json({
    message: "Hello, #{@who}!"
  })
end
```

> `.json` is a special DSL (*macro*), which defines the way this view will be rendered to JSON. Read more in [Onyx::REST::View](https://api.onyxframework.org/rest/Onyx/REST/View.html) docs.

Now we can use this view in our action:

{{< highlight crystal "hl_lines=5" >}}
struct Actions::Hello
  include Onyx::REST::Action

  def call
    return Views::Hello.new("Onyx")
  end
end
{{< / highlight >}}

{{< highlight crystal "hl_lines=3 8" >}}
require "onyx/rest"

require "./views/**"
require "./actions/**"

Onyx.get "/", Actions::Hello

Onyx.render(:json)
Onyx.listen
{{< / highlight >}}

The response should now return a JSON string:

```sh
> curl http://localhost:5000
{"message":"Hello, Onyx!"}
```

### Adding query params

Onyx::REST::Action has a powerful `.params` DSL which allows to define strongly-typed parameters. Currently the endpoint always returns `"Hello, Onyx!"`. Let's change it with a dynamic query parameter:

{{< highlight crystal "hl_lines=4-8 11-12" >}}
struct Actions::Hello
  include Onyx::REST::Action

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
> curl http://localhost:5000
{"message":"Hello, Onyx!"}
> curl http://localhost:5000?who=Crystal
{"message":"Hello, Crystal!"}
```

That's all for part I of this tutorial. You've learned how to create REST API endpoints with separate business and rendering layers.

[Continue to part 2 →](/posts/creating-json-apis-with-onyx-part-2)
