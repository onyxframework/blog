---
title: "v0.4.0 Released"
date: 2019-04-20T20:17:07+03:00
author: Vlad Faust
tags: [Crystal, Onyx Framework]
keywords: [Crystal, Onyx Framework]
---

# Onyx Framework v0.4.0 released!

The latest release brings Crystal 0.28 support, brand new server testing API, fresh [EDA update](https://github.com/onyxframework/eda/releases/tag/v0.3.0) and better modularity. Many bugs were fixed and _some_ were added.

<!--more-->

## General changes

- **(Breaking)** Onyx now supports [Crystal version 0.28+](https://crystal-lang.org/2019/04/17/crystal-0.28.0-released.html)
- **(Breaking)** Module methods are now namespaced:

Before:

```crystal
Onyx.get "/", &.response.puts("Hello!")
Onyx.listen
```

After:

```crystal
Onyx::HTTP.get "/", &.response.puts("Hello!")
Onyx::HTTP.listen
```

This change also applies both to Onyx::SQL and Onyx::EDA.

- CI improvements – now every onyx\* commit triggers many dependant repository builds, including [crystalworld](https://github.com/vladfaust/crystalworld) and [onyx-todo-json-api](https://github.com/vladfaust/onyx-todo-json-api), to catch bugs earlier

## Onyx::HTTP changes

> The appropriate version is now **~> 0.8.0**.

- Added `Onyx::HTTP.on` method to build tree-like structures ([API docs](https://api.onyxframework.com/http/Onyx/HTTP/Middleware/Router.html#on%28namespace%3AString%3F%3Dnil%2C%26block%3Aself-%3E%29-instance-method)):

Before:

```crystal
Onyx.post "/users/", Endpoint::User::Create
Onyx.get "/users/", Endpoint::User::Index
Onyx.get "/users/:id", Endpoint::User::Get
```

After:

```crystal
Onyx::HTTP.on "/users" do |r|
  r.post "/", Endpoint::User::Create
  r.get "/", Endpoint::User::Index
  r.get "/:id", Endpoint::User::Get
end
```

- Added HTTP spec helpers ([API docs](https://api.onyxframework.com/onyx/Onyx/HTTP/Spec.html)):

```crystal
require "onyx/http/spec"
require "../src/server"

it do
  response = Onyx::HTTP::Spec.get("/")
  response.assert(200, "Hello, Onyx!")
end
```

See the full changelog at [Onyx::HTTP](https://github.com/onyxframework/http/releases) and [Onyx](https://github.com/onyxframework/onyx/releases) releases.

## Onyx::SQL changes

> The appropriate version is now **~> 0.8.0**.

- Added `Onyx::SQL.transaction` method to wrap a database transaction (see its [API docs](https://api.onyxframework.com/onyx/Onyx/SQL.html#transaction%28%26block%3ADB%3A%3ATransaction-%3E%29-class-method)):

```crystal
Onyx::SQL.transaction do
  # Repo uses the transaction connection now
  Onyx::SQL.query(User.where(status: :active))
end
```

- Added BulkQuery allowing to insert many models at once ([docs](https://docs.onyxframework.com/sql/query.html#enumerable-shortcuts)):

```crystal
Onyx::SQL.query(users.insert)
```

See the full changelog at [Onyx::SQL](https://github.com/onyxframework/sql/releases) and [Onyx](https://github.com/onyxframework/onyx/releases) releases.

## Onyx::EDA changes

> The appropriate version is now **~> 0.3.0**.

Onyx::EDA got a sweet new release [v0.3.0](https://github.com/onyxframework/eda/releases/tag/v0.3.0), which is basically an overhaul with many new features, including filtering and awaiting for events. See it in the [repo README](https://github.com/onyxframework/eda).
