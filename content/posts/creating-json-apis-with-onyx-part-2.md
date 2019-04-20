---
title: "Creating JSON APIs with Onyx. Part 2 — CRUD"
date: 2019-02-24T19:22:25+03:00
lastmod: 2019-04-20
draft: false
author: Vlad Faust
tags: [Crystal, Onyx Framework]
keywords: [Crystal, Onyx Framework, Tutorial]
---

In the second part of the tutorial we are going to create CRUD endpoints for our Todo list JSON API. You should have the code from [the first part](/posts/creating-json-apis-with-onyx-part-1/) available. You can also clone it from [GitHub](https://github.com/vladfaust/onyx-todo-json-api/tree/part-1).

<!--more-->

## Tutorial contents

1. [Part 1 — The First Endpoint](/posts/creating-json-apis-with-onyx-part-1)
2. Part 2 — CRUD (*this article*)

---

We will be using [Onyx::SQL](https://onyxframework.org/sql) as an ORM and [PostgreSQL](https://www.postgresql.org/) as a database (although it's possible to use another SQL database with slight changes to the code).

**Table of contents:**

1. [Creating a migration](#creating-a-migration)
2. [Env files](#env-files)
3. [Running the migration](#running-the-migration)
4. [Defining a model](#defining-a-model)
5. [Adding the Create endpoint](#adding-the-create-endpoint)
6. [Adding the Index endpoint](#adding-the-index-endpoint)
7. [Adding the Get endpoint](#adding-the-get-endpoint)
8. [Adding the Update endpoint](#adding-the-update-endpoint)
9. [Adding the Delete endpoint](#adding-the-delete-endpoint)
10. [Next steps](#next-steps)

### Creating a migration

As we have an SQL database as a back-end, we'll need a database table to store the Todo list items. Make sure you're familiar with PostgreSQL and have a running server instance on your machine.

> You can quickly run a PostgreSQL server with the [Docker image](https://hub.docker.com/_/postgres) and administrate it with [PGAdmin](https://www.pgadmin.org/).

Create a new migration file at `db/migrations/001_create_items.sql`:

{{< highlight plaintext "hl_lines=2-4" >}}
.
└── db
    └── migrations
        └── 001_create_items.sql
{{< / highlight >}}

```sql
-- +migrate up
CREATE TABLE items (
  id          SERIAL      PRIMARY KEY,
  completed   BOOLEAN     NOT NULL  DEFAULT false,
  content     TEXT        NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL  DEFAULT now(),
  updated_at  TIMESTAMPTZ
);

CREATE INDEX items_completed  ON items (completed);
CREATE INDEX items_created_at ON items (created_at);

-- +migrate down
DROP TABLE items;
```

Special `-- +migrate` comments are part of [migrate.cr](https://github.com/vladfaust/migrate.cr) shard. To actually run a migration, you must have this shard installed, as seen in the following section.

### Env files

As you have an SQL dependency, your application now relies on some environment variables, such as `DATABASE_URL`. It's a good idea to store them in `.env` files.

Create a new `.env.development.local` file in the root of your project and fill it with your actual `DATABASE_URL` variable:

{{< highlight plaintext "hl_lines=2-4" >}}
.
└── .env.development.local
{{< / highlight >}}

```plaintext
DATABASE_URL=postgres://postgres:postgres@localhost:5432/todo-onyx
```

The code responsible for loading environment variables from `.env` files is included into all Onyx components by default.

### Running the migration

To run migrations from within the application, the [migrate.cr](https://github.com/vladfaust/migrate.cr) and [pg](https://github.com/will/crystal-pg) shards must be installed. Modify your `shard.yml` file with the following additions:

{{< highlight yaml "hl_lines=8-16" >}}
dependencies:
  onyx:
    github: onyxframework/onyx
    version: ~> 0.4.0
  onyx-http:
    github: onyxframework/http
    version: ~> 0.8.0
  onyx-sql:
    github: onyxframework/sql
    version: ~> 0.8.0
  pg:
    github: will/crystal-pg
    version: ~> 0.16.0
  migrate:
    github: vladfaust/migrate.cr
    version: ~> 0.4.0
{{< / highlight >}}

Run `shards install` to install the freshly added dependencies.

To run tasks in Crystal, [Cake](https://github.com/axvm/cake) is used, which is just like Rake. Find installation instructions for Cake at <https://github.com/axvm/cake>. To continue, please make sure you've properly installed the binary with `cake -v` command.

Now create a `Cakefile` at the root of the project. It is like Rakefile, but for Crystal:

{{< highlight plaintext "hl_lines=2-4" >}}
.
└── Cakefile
{{< / highlight >}}

```crystal
require "pg"
require "migrate"

require "onyx"
require "onyx/db"

desc "Migrate database to the latest version"
task :dbmigrate do
  migrator = Migrate::Migrator.new(
    Onyx.db,
    Onyx.logger
  )

  migrator.to_latest
end

desc "Reset database to zero and then to the latest version"
task :dbredo do
  migrator = Migrate::Migrator.new(
    Onyx.db,
    Onyx.logger
  )

  migrator.redo
end
```

Run the migration with `cake db:migrate`:

```sh
> cake db:migrate
 INFO [20:29:28.661] Successfully migrated from version 0 to 1 in 72.377ms
```

Now you have a table in the database, it's time to link it to the application code.

### Defining a model

Add a new file at `src/models/item.cr`:

{{< highlight plaintext "hl_lines=2-3" >}}
└── src
    ├── models
    │   └── item.cr
    └── server.cr
{{< / highlight >}}

```crystal
require "pg"
require "onyx/sql"

class Models::Item
  include Onyx::SQL::Model

  schema items do
    pkey id : Int32
    type completed : Bool, not_null: true, default: true
    type content : String, not_null: true
    type created_at : Time, not_null: true, default: true
    type updated_at : Time
  end
end
```

That's all you need to define a model mapping!

### Adding the Create endpoint

Add a new file at `src/endpoints/items/create.cr` and put the following code inside it.

{{< highlight plaintext "hl_lines=3-4" >}}
└── src
    └── endpoints
        └── items
            └── create.cr
{{< / highlight >}}

```crystal
struct Endpoints::Items::Create
  include Onyx::HTTP::Endpoint

  params do
    # Will attempt to parse JSON params even if
    # "Content-Type" header is not "application/json"
    json require: true do
      type content : String
    end
  end

  def call
    # Insert the model into the database
    #

    item = Models::Item.new(content: params.json.content)
    Onyx::SQL.exec(item.insert)

    # Return the success status
    #

    status(201)
  end
end
```

You should then make Onyx aknowledge of the new endpoint:

{{< highlight crystal "hl_lines=3 8" >}}
require "onyx/http"

require "./models/**"
require "./views/**"
require "./endpoints/**"

Onyx::HTTP.get "/", Endpoints::Hello
Onyx::HTTP.post "/items", Endpoints::Items::Create

Onyx::HTTP.listen
{{< / highlight >}}

From now on, the application **requires** `DATABASE_URL` environment variable to be set in runtime (you should have already defined it in the `.env.development.local` file). Run the server with the following command:

```sh
> crystal src/server.cr
```

And make a POST request to the new endpoint:

```sh
> curl -X POST -d '{"content": "Learn Crystal"}' -v http://127.0.0.1:5000/items
```

You should see the valid status in the response:

```sh
< HTTP/1.1 201 Created
```

Currently the response doesn't contain the created item itself. Let's add a corresponding view for it.

{{< highlight plaintext "hl_lines=3" >}}
└── src
    └── views
        └── item.cr
{{< / highlight >}}

```crystal
struct Views::Item
  include Onyx::HTTP::View

  def initialize(@item : Models::Item)
  end

  json(
    id: @item.id,
    completed: @item.completed,
    content: @item.content,
    createdAt: @item.created_at,
    updatedAt: @item.updated_at
  )
end
```

Then modify the Create endpoint so it returns the view. Previously we were using `Onyx.exec` to execute an SQL query and not expect anything in return. Now we use `Onyx.query` to actually query the database with `INSERT ... RETURNING` query:

{{< highlight crystal "hl_lines=5 11" >}}
    # Insert the model into the database
    #

    item = Models::Item.new(content: params.json.content)
    item = Onyx::SQL.query(item.insert.returning("*")).first

    # Return the success status
    #

    status(201)
    return Views::Item.new(item)
  end
end
{{< / highlight >}}

The endpoint should now return the freshly created item:

```sh
> curl -X POST -d '{"content": "Learn Onyx"}' -v http://127.0.0.1:5000/items
...
< HTTP/1.1 201 Created
< Content-Type: application/json; charset=utf-8
< Content-Length: 110
...
{"item":{"id":2,"completed":false,"content":"Learn Onyx","createdAt":"2019-02-22T21:49:33Z","updatedAt":null}}
```

You may also notice the repository logs of your server process:

```sh
DEBUG [00:42:55.283] [postgresql] INSERT INTO items (content, updated_at) VALUES ($1, $2) RETURNING *
DEBUG [00:42:55.286] 2.731ms
```

### Adding the Index endpoint

Now we want to list all the items we have. To do so, create a new Index endpoint at `src/endpoints/items/index.cr` and a new view at `src/views/items.cr`. The code below is pretty self-explainatory.

{{< highlight plaintext "hl_lines=5 8" >}}
└── src
    ├── endpoints
    │   └── items
    │       ├── create.cr
    │       └── index.cr
    └── views
        ├── item.cr
        └── items.cr
{{< / highlight >}}

```crystal
# src/endpoints/items/index.cr
#

struct Endpoints::Items::Index
  include Onyx::HTTP::Endpoint

  def call
    items = Onyx::SQL.query(Models::Item.all)
    return Views::Items.new(items)
  end
end
```

```crystal
# src/views/items.cr
#

struct Views::Items
  include Onyx::HTTP::View

  def initialize(@items : Enumerable(Models::Item))
  end

  json items: @items.map { |i| Views::Item.new(i) }
end
```

Finally add the new endpoint to `src/server.cr`:

{{< highlight crystal "hl_lines=3" >}}
Onyx::HTTP.get "/", Endpoints::Hello
Onyx::HTTP.post "/items", Endpoints::Items::Create
Onyx::HTTP.get "/items", Endpoints::Items::Index
{{< / highlight >}}

Run the server and validate the endpoint with curl:

```sh
> curl http://127.0.0.1:5000/items
{"items":[{"id":1,"completed":false,"content":"Learn Crystal","createdAt":"2019-02-22T22:11:33Z","updatedAt":null},{"id":2,"completed":false,"content":"Learn Onyx","createdAt":"2019-02-22T22:11:39Z","updatedAt":null}]}
```

### Adding the Get endpoint

Get Endpoint is very simple, you don't need a new View for it:

{{< highlight plaintext "hl_lines=5" >}}
└── src
    └── endpoints
        └── items
            ├── create.cr
            ├── get.cr
            └── index.cr
{{< / highlight >}}

```crystal
struct Endpoints::Items::Get
  include Onyx::HTTP::Endpoint

  params do
    path do
      type id : Int32
    end
  end

  errors do
    # Return 404 if item is not found
    type ItemNotFound(404)
  end

  def call
    item = Onyx::SQL.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    return Views::Item.new(item)
  end
end
```

{{< highlight crystal "hl_lines=4" >}}
Onyx::HTTP.get "/", Endpoints::Hello
Onyx::HTTP.post "/items", Endpoints::Items::Create
Onyx::HTTP.get "/items", Endpoints::Items::Index
Onyx::HTTP.get "/items/:id", Endpoints::Items::Get
{{< / highlight >}}

Don't forget to test it with curl:

```sh
> curl http://127.0.0.1:5000/items/1
{"item":{"id":1,"completed":false,"content":"Learn Crystal","createdAt":"2019-02-22T22:11:33Z","updatedAt":null}}
> curl http://127.0.0.1:5000/items/0
{"error":{"class":"ItemNotFound","message":"Item Not Found","code":404}}
```

### Adding the Update endpoint

To be able to update created items, you'll need a brand new Update endpoint:

{{< highlight plaintext "hl_lines=7" >}}
└── src
    └── endpoints
        └── items
            ├── create.cr
            ├── get.cr
            ├── index.cr
            └── update.cr
{{< / highlight >}}

```crystal
struct Endpoints::Items::Update
  include Onyx::HTTP::Endpoint

  params do
    path do
      type id : Int32
    end

    json require: true do
      type completed : Bool?
      type content : String?
    end
  end

  # Define HTTP errors
  #

  errors do
    # Return 400 if there is nothing to update
    type NothingToUpdate(400)

    # Return 404 if item is not found
    type ItemNotFound(404)
  end

  def call
    # Validate the request
    #

    raise NothingToUpdate.new if params.json.content.nil? && params.json.completed.nil?

    # Fetch the item from DB
    #

    item = Onyx::SQL.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    # Create a new changeset with a snapshot of actual item's values
    #

    changeset = item.changeset

    if content = params.json.content
      changeset.update(content: content)
    end

    unless params.json.completed.nil?
      changeset.update(completed: params.json.completed)
    end

    # Halt if there are no actual changes
    raise NothingToUpdate.new if changeset.empty?

    # Update the updated_at field
    changeset.update(updated_at: Time.now)

    # Update the item with modified changeset returning itself
    #

    item = Onyx::SQL.query(item.update(changeset).returning(Models::Item)).first
    return Views::Item.new(item)
  end
end
```

{{< highlight crystal "hl_lines=5" >}}
Onyx::HTTP.get "/", Endpoints::Hello
Onyx::HTTP.post "/items", Endpoints::Items::Create
Onyx::HTTP.get "/items", Endpoints::Items::Index
Onyx::HTTP.get "/items/:id", Endpoints::Items::Get
Onyx::HTTP.patch "/items/:id", Endpoints::Items::Update
{{< / highlight >}}

Looks like you're doing great with Onyx! Let's update the corresponding item:

```sh
> curl -X PATCH -d '{"completed": true}' http://127.0.0.1:5000/items/2
{"item":{"id":2,"completed":true,"content":"Learn Onyx","createdAt":"2019-02-22T22:11:39Z","updatedAt":null}}
```

### Adding the Delete endpoint

As you've completed the item, you may want to delete it. For this, create a new Delete endpoint at `src/endpoints/items/delete.cr`:

{{< highlight plaintext "hl_lines=5" >}}
└── src
    └── endpoints
        └── items
            ├── create.cr
            ├── delete.cr
            ├── get.cr
            ├── index.cr
            └── update.cr
{{< / highlight >}}

```crystal
struct Endpoints::Items::Delete
  include Onyx::HTTP::Endpoint

  params do
    path do
      type id : Int32
    end
  end

  errors do
    # Return 404 when item is not found
    type ItemNotFound(404)
  end

  def call
    item = Onyx::SQL.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    Onyx::SQL.exec(item.delete)
    status(202)
  end
end
```

{{< highlight crystal "hl_lines=6" >}}
Onyx::HTTP.get "/", Endpoints::Hello
Onyx::HTTP.post "/items", Endpoints::Items::Create
Onyx::HTTP.get "/items", Endpoints::Items::Index
Onyx::HTTP.get "/items/:id", Endpoints::Items::Get
Onyx::HTTP.patch "/items/:id", Endpoints::Items::Update
Onyx::HTTP.delete "/items/:id", Endpoints::Items::Delete
{{< / highlight >}}

Delete the item:

```sh
> curl -X DELETE http://127.0.0.1:5000/items/2
< HTTP/1.1 202 Accepted
```

### Beautifying the routes

You can make the routes a bit more shiny using a tree-like `on` syntax:

```crystal
Onyx::HTTP.get "/", Endpoints::Hello

Onyx::HTTP.on "/items" do |r|
  r.get "/", Endpoints::Items::Create
  r.get "/", Endpoints::Items::Index
  r.get "/:id", Endpoints::Items::Get
  r.patch "/:id", Endpoints::Items::Update
  r.delete "/:id", Endpoints::Items::Delete
end
```

Congratulations! You have your own full-featured JSON REST API in Onyx! 🎉 The full source code of this part is available at [GitHub](https://github.com/vladfaust/onyx-todo-json-api/tree/part-2).

## Next steps

You should now be ready to dive into the Onyx documentation at [docs.onyxframework.org](https://docs.onyxframework.org). If you have questions left or just want to chat, join the framework's [Gitter](https://gitter.im/onyxframework) and follow the [Twitter](https://twitter.com/onyxframework).
