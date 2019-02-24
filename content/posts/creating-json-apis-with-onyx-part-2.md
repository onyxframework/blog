---
title: "Creating JSON APIs with Onyx. Part 2 — CRUD"
date: 2019-02-24T19:22:25+03:00
draft: false
author: Vlad Faust
tags: [Crystal, Onyx Framework]
keywords: [Crystal, Onyx Framework, Tutorial]
---

In the second part of the tutorial we're going to create CRUD actions for our Todo list JSON API. You should have the code from [the first part](/posts/creating-json-apis-with-onyx-part-1/) available.

<!--more-->

## Tutorial contents

1. [Part 1 — The First Endpoint](/posts/creating-json-apis-with-onyx-part-1)
2. Part 2 — CRUD (*this article*)

---

We will be using [`Onyx::SQL`](https://onyxframework.org/sql) as an ORM and [PostgreSQL](https://www.postgresql.org/) as a database (although it's possible to use another SQL database with slight changes to the code).

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

And add the code which would load from this file in `src/server.cr`:

{{< highlight crystal "hl_lines=1" >}}
require "onyx/env"
require "onyx/rest"

require "./views/**"
require "./actions/**"
{{< / highlight >}}

### Running the migration

To run migrations from within the application, the [migrate.cr](https://github.com/vladfaust/migrate.cr) and [pg](https://github.com/will/crystal-pg) shards must be installed. Modify your `shard.yml` file with the following additions:

{{< highlight yaml "hl_lines=8-16" >}}
dependencies:
  onyx:
    github: onyxframework/onyx
    version: ~> 0.1.2
  onyx-rest:
    github: onyxframework/rest
    version: ~> 0.6.3
  pg:
    github: will/crystal-pg
    version: ~> 0.15.0
  migrate:
    github: vladfaust/migrate.cr
    version: ~> 0.4.1
{{< / highlight >}}

Run `shards install` to actually install the freshly added dependencies.

To run tasks in Crystal, [Cake](https://github.com/axvm/cake) is used, which is just like Rake. Find installation instructions for Cake at <https://github.com/axvm/cake>. To continue, please make sure you've properly installed the binary with `cake -v` command.

Now create a `Cakefile` at the root of the project. It's like Rakefile, but for Crystal:

{{< highlight plaintext "hl_lines=2-4" >}}
.
└── Cakefile
{{< / highlight >}}

```crystal
require "pg"
require "migrate"

require "onyx/env"
require "onyx/db"
require "onyx/logger"

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

```plaintext
> cake db:migrate
 INFO [20:29:28.661 #31308] Successfully migrated from version 0 to 1 in 72.377ms
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
    type completed : Bool, default: true
    type content : String
    type created_at : Time, default: true
    type updated_at : Time
  end
end
```

That's all you need to define a model mapping!

### Adding the Create endpoint

Add a new file at `src/actions/items/create.cr` and put the following code inside it.

{{< highlight plaintext "hl_lines=3-4" >}}
└── src
    └── actions
        └── items
            └── create.cr
{{< / highlight >}}

```crystal
struct Actions::Items::Create
  include Onyx::REST::Action

  params do
    json do
      type content : String
    end
  end

  # Define REST errors
  #

  errors do
    # Return 400 if a request doesn't have a
    # "Content-Type" header equal to "application/json"
    type JSONContentTypeRequired(400)
  end

  def call
    # Validate the request payload
    #

    json = params.json
    raise JSONContentTypeRequired.new unless json

    # Insert the model into the database
    #

    item = Models::Item.new(content: json.content)
    Onyx.exec(item.insert)

    # Return the success status
    #

    status(201)
  end
end
```

You should then make Onyx aknowledge of the new action:

> I cannot highlight the line in this block due to a [bug](https://github.com/alecthomas/chroma/issues/232) in Chroma code highlighting


```crystal
require "onyx/rest"

require "./models/**"
require "./views/**"
require "./actions/**"

Onyx.get "/", Actions::Hello
Onyx.post "/items", Actions::Items::Create # << New line

Onyx.render(:json)
Onyx.listen
```

From now on, the application **requires** `DATABASE_URL` environment variable to be set in runtime (we defined it in the `.env.development.local` file). Run the server with the following command:

```sh
> crystal src/server.cr
```

And make a POST request to the new endpoint:

```sh
> curl -X POST -d '{"content": "Learn Crystal"}' -H 'Content-Type: application/json' -v http://localhost:5000/items
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
  include Onyx::REST::View

  def initialize(@item : Models::Item)
  end

  json do
    # At this moment we're in the root of the JSON document
    #

    object do                  # Add `{`
      field "item" do          # Add `"item":`
        object do              # Add another `{`
          field "id", @item.id # Add `"id":1`, ditto below
          field "completed", @item.completed
          field "content", @item.content
          field "createdAt", @item.created_at
          field "updatedAt", @item.updated_at?
        end # Close object with `}`
      end
    end # ditto
  end
end
```

Then modify the Create action so it returns the view. Previously we were using `Onyx.exec` to execute an SQL query and not expect anything in return. Now we use `Onyx.query` to actually query the database with `INSERT ... RETURNING` query:

{{< highlight crystal "hl_lines=5 11" >}}
    # Insert the model into the database
    #

    item = Models::Item.new(content: json.content)
    item = Onyx.query(item.insert.returning("*")).first

    # Return the success status
    #

    status(201)
    return Views::Item.new(item)
  end
end
{{< / highlight >}}

The endpoint should now return the freshly created item:

```sh
> curl -X POST -d '{"content": "Learn Onyx"}' -H 'Content-Type: application/json' -v http://localhost:5000/items
...
< HTTP/1.1 201 Created
< Content-Type: application/json; charset=utf-8
< Content-Length: 110
...
{"item":{"id":2,"completed":false,"content":"Learn Onyx","createdAt":"2019-02-22T21:49:33Z","updatedAt":null}}
```

You may also notice the repository logs of your server process:

```plaintext
DEBUG [00:42:55.283 #15402] [postgresql] INSERT INTO items (content, updated_at) VALUES ($1, $2) RETURNING *
DEBUG [00:42:55.286 #15402] 2.731ms
```

### Adding the Index endpoint

Now we want to list all the items we have. To do so, create a new Index action at `src/actions/items/index.cr` and a new view at `src/views/items.cr`. The code below is pretty self-explainatory.

{{< highlight plaintext "hl_lines=5 8" >}}
└── src
    ├── actions
    │   └── items
    │       ├── create.cr
    │       └── index.cr
    └── views
        ├── item.cr
        └── items.cr
{{< / highlight >}}

```crystal
# src/actions/items/index.cr
#

struct Actions::Items::Index
  include Onyx::REST::Action

  def call
    items = Onyx.query(Models::Item.all)
    return Views::Items.new(items)
  end
end
```

```crystal
# src/views/items.cr
#

struct Views::Items
  include Onyx::REST::View

  def initialize(@items : Enumerable(Models::Item))
  end

  json do
    object do
      field "items" do
        array do
          @items.each do |item|
            # Note the new `true` argument
            # `itself` refers to the current JSON object
            Views::Item.new(item, true).to_json(itself)
          end
        end
      end
    end
  end
end
```

Also modify the `Views::Item` view, so we it becomes "embeddable":

{{< highlight crystal "hl_lines=4 9-14 22-24" >}}
struct Views::Item
  include Onyx::REST::View

  def initialize(@item : Models::Item, @nested = false) # Add `nested` arg
  end

  json do
    object do # Add `{`
      unless @nested
        # If we're **not** within a nested JSON, then
        # begin a new object with `"item":{`
        string "item"
        start_object
      end

      field "id", @item.id # Add `"id":1`, ditto below
      field "completed", @item.completed
      field "content", @item.content
      field "createdAt", @item.created_at
      field "updatedAt", @item.updated_at?

      unless @nested
        end_object
      end
    end # ditto
  end
end
{{< / highlight >}}

Finally add the new action to `src/server.cr`:

```crystal
Onyx.get "/", Actions::Hello
Onyx.post "/items", Actions::Items::Create
Onyx.get "/items", Actions::Items::Index # << This
```

Run the server and validate the endpoint with curl:

```sh
> curl http://localhost:5000/items
{"items":[{"id":1,"completed":false,"content":"Learn Crystal","createdAt":"2019-02-22T22:11:33Z","updatedAt":null},{"id":2,"completed":false,"content":"Learn Onyx","createdAt":"2019-02-22T22:11:39Z","updatedAt":null}]}
```

### Adding the Get endpoint

Get Action is very simple, you don't need a new View for it:

{{< highlight plaintext "hl_lines=5" >}}
└── src
    └── actions
        └── items
            ├── create.cr
            ├── get.cr
            └── index.cr
{{< / highlight >}}

```crystal
struct Actions::Items::Get
  include Onyx::REST::Action

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
    item = Onyx.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    return Views::Item.new(item)
  end
end
```

```crystal
Onyx.get "/", Actions::Hello
Onyx.post "/items", Actions::Items::Create
Onyx.get "/items", Actions::Items::Index
Onyx.get "/items/:id", Actions::Items::Get # << New line
```

Don't forget to test it with curl:

```sh
> curl http://localhost:5000/items/1
{"item":{"id":1,"completed":false,"content":"Learn Crystal","createdAt":"2019-02-22T22:11:33Z","updatedAt":null}}
> curl http://localhost:5000/items/0
{"error":{"class":"ItemNotFound","message":"Item Not Found","code":404}}
```

### Adding the Update endpoint

To be able to update created items, you'll need a brand new Update action:

{{< highlight plaintext "hl_lines=7" >}}
└── src
    └── actions
        └── items
            ├── create.cr
            ├── get.cr
            ├── index.cr
            └── update.cr
{{< / highlight >}}

```crystal
struct Actions::Items::Update
  include Onyx::REST::Action

  params do
    path do
      type id : Int32
    end

    json do
      type completed : Bool?
      type content : String?
    end
  end

  # Define REST errors
  #

  errors do
    # Return 400 if a request doesn't have a
    # "Content-Type" header equal to "application/json"
    type JSONContentTypeRequired(400)

    # Return 400 if there is nothing to update
    type NothingToUpdate(400)

    # Return 404 if item is not found
    type ItemNotFound(404)
  end

  def call
    # Validate the request
    #

    json = params.json
    raise JSONContentTypeRequired.new unless json
    raise NothingToUpdate.new unless json.content || !json.completed.nil?

    # Fetch the item from DB
    #

    item = Onyx.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    # Create a new changeset with a snapshot of actual item's values
    #

    changeset = item.changeset

    if content = json.content
      changeset.content = content
    end

    unless json.completed.nil?
      changeset.completed = json.completed
    end

    # Halt if there are no actual changes
    raise NothingToUpdate.new if changeset.empty?

    # Update the item with modified changeset returning itself
    #

    item = Onyx.query(item.update(changeset).returning(Models::Item)).first
    return Views::Item.new(item)
  end
end
```

```crystal
Onyx.get "/", Actions::Hello
Onyx.post "/items", Actions::Items::Create
Onyx.get "/items", Actions::Items::Index
Onyx.get "/items/:id", Actions::Items::Get
Onyx.put "/items/:id", Actions::Items::Update # << Added
```

Looks like you're doing great with Onyx! Let's update the corresponding item:

```sh
> curl -X PUT -d '{"completed": true}' -H 'Content-Type: application/json' http://localhost:5000/items/2
{"item":{"id":2,"completed":true,"content":"Learn Onyx","createdAt":"2019-02-22T22:11:39Z","updatedAt":null}}
```

### Adding the Delete endpoint

As you've completed the item, you may want to delete it. For this, create a new Delete action at `src/actions/items/delete.cr`:

{{< highlight plaintext "hl_lines=5" >}}
└── src
    └── actions
        └── items
            ├── create.cr
            ├── delete.cr
            ├── get.cr
            ├── index.cr
            └── update.cr
{{< / highlight >}}

```crystal
struct Actions::Items::Delete
  include Onyx::REST::Action

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
    item = Onyx.query(Models::Item.where(id: params.path.id)).first?
    raise ItemNotFound.new unless item

    Onyx.exec(item.delete)
    status(202)
  end
end
```

```crystal
Onyx.get "/", Actions::Hello
Onyx.post "/items", Actions::Items::Create
Onyx.get "/items", Actions::Items::Index
Onyx.get "/items/:id", Actions::Items::Get
Onyx.put "/items/:id", Actions::Items::Update
Onyx.delete "/items/:id", Actions::Items::Delete # << Finally, this line
```

Delete the item:

```sh
> curl -X DELETE http://localhost:5000/items/2
< HTTP/1.1 202 Accepted
```

Congratulations! You now have a full-featured JSON REST API in Crystal! 🎉
