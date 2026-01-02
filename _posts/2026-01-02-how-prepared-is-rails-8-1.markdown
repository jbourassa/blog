---
layout: post
title:  "How prepared is Rails 8.1?"
date:   2026-01-02 12:21:55 -0500
---

At [work](https://missiveapp.com/), we tried to get all SQL queries for a given
transaction to use prepared statements. This was my first time using prepared
statements in Rails, and was surprised by what Rails does not prepare.

Let's dive into what Rails 8.1 prepares, what it does not, and what we might do
about it.

## Identifying prepared statements
First, we need a way to identify which SQL query runs as a prepared
statement. With Postgres, we have it easy: we can use the built-in
`sql.active_record` instrumentation hook and read the [`:statement_name` key](https://github.com/rails/rails/blob/v8.1.1/guides/source/active_support_instrumentation.md?plain=1#L554).

```ruby
ActiveSupport::Notifications.subscribe("sql.active_record") do |event|
  next if event.payload[:name] == "SCHEMA"

  if event.payload[:statement_name].nil?
    puts "❌ NOT PREPARED: #{event.payload[:sql]}"
  end
end
```

For other adapters, [Ben Sheldon's trick](https://island94.org/2024/03/rails-active-record-will-it-bind)
of using `ActiveRecord::Connection#to_sql_and_binds` works for `ActiveRecord::Relation`,
but falls short for methods that do not return relations like `#pluck` or `#count`.

## State of things

Using the above instrumentation, let's see which queries get prepared.

### Where

`#where` gets prepared when using a Hash or sending any `?`-bound params, but
not when sending a plain String, or for `IN` clauses.
```ruby
Post.where(title: "Hello world").load
# ↳ ✅ SELECT "posts".* FROM "posts" WHERE "posts"."title" = $1

Post.where("title = ?", "Hello world").load
# ↳ ✅ SELECT "posts".* FROM "posts" WHERE (title = $1)

Post.where("title = 'Hello world'").load
# ↳ ❌ SELECT "posts".* FROM "posts" WHERE (title = 'Hello world')

Post.where("public IS NOT TRUE").load
# ↳ ❌ SELECT "posts".* FROM "posts" WHERE (public IS NOT TRUE)

Post.where(title: %w[foo bar]).load
# ↳ ❌ SELECT "posts".* FROM "posts" WHERE "posts"."title" IN ($1, $2)
```

### Select

`#select` gets prepared when each argument matches exactly a column name or
the `table.column` format. Anything else does not.

Same logic applies to the various `ActiveRecord::Calculations` methods (e.g. `#sum`).

```ruby
Post.select('id').load
Post.select(:id).load
Post.select('posts.id').load
# ↳ ✅ SELECT "posts"."id" FROM "posts"

Post.select('"posts"."id"').load
# ↳ ❌ SELECT "posts"."id" FROM "posts"

Post.select('id, views').load
# ↳ ❌ SELECT id, views FROM "posts"

Post.select('COALESCE(public, false) AS public').load
# ↳ ❌ SELECT COALESCE(public, false) AS public FROM "posts"
```

### Order

`#order` gets prepared only when using Symbol or Hash-style `col: :direction`.

```ruby
Post.order(:views).load
Post.order(views: :asc).load
# ↳ ✅ SELECT "posts".* FROM "posts" ORDER BY "posts"."views" ASC

Post.order('views').load
# ↳ ❌ SELECT "posts".* FROM "posts" ORDER BY views
```

### Joins

Joins get prepared when using associations, not when using SQL fragments as Strings.

```ruby
Post.joins(:author).load
# ↳ ✅ SELECT "posts".* FROM "posts" INNER JOIN "authors" ON "authors"."id" = "posts"."author_id"

Post.joins("LEFT JOIN authors ON posts.author_id = authors.id").load
# ↳ ❌ SELECT "posts".* FROM "posts" LEFT JOIN authors ON posts.author_id = authors.id
```

### Subqueries

Subqueries get prepared too, except when using the `IN (?)` form.

```ruby
Post.where(author_id: Author.select(:id).where("id > ?", 1)).load
# ↳ ✅ SELECT "posts".* FROM "posts" WHERE "posts"."author_id" IN (SELECT "authors"."id" FROM "authors" WHERE (id > $1))

Post.where("author_id IN (?)", Author.select(:id).where("id > ?", 1)).load
# ↳ ❌ SELECT "posts".* FROM "posts" WHERE (author_id IN (SELECT "authors"."id" FROM "authors" WHERE (id > 1)))
```

### Misc

`#any?` and `#count` do not get prepared.
```ruby
Post.any?
# ↳ ❌ SELECT 1 AS one FROM "posts" LIMIT $1
Post.count
# ↳ ❌ SELECT COUNT(*) FROM "posts"
```

`#distinct` is fine.
```ruby
Post.distinct.pluck(:id)
# ↳ ✅ SELECT DISTINCT "posts"."id" FROM "posts"
```

## But why?

ActiveRecord walks the tree of Arel nodes. Each node type can set the resulting
query as not-preparable. [SQL string literal](https://github.com/rails/rails/blob/624fe3cdb9ab774ff598af29f408425178da6677/activerecord/lib/arel/visitors/to_sql.rb#L767)
and [`IN`](https://github.com/rails/rails/blob/624fe3cdb9ab774ff598af29f408425178da6677/activerecord/lib/arel/visitors/to_sql.rb#L594) / `NOT IN` are the only offenders:

```ruby
def visit_Arel_Nodes_SqlLiteral(o, collector)
  collector.preparable = false # <-- the culprit!
  collector.retryable &&= o.retryable
  collector << o.to_s
end
```

But then again, why? I don't know for sure, but I bet it's because Strings can
contain arbitrary arguments thus cause the number of prepared statements
to balloon. Contrived example: `Post.where("id = #{Integer(params[:id])}")`.

## What might we do about it?

Since Rails 7.2 ([#51336](https://github.com/rails/rails/pull/51336)),
[`Arel.sql`](https://api.rubyonrails.org/v8.1.1/classes/Arel.html#method-c-sql)
accepts a `retryable:` bool kwarg. Knowing whether a query is retryable is
similar to knowing whether it is preparable; a similar implementation would
make sense.

`Arel.sql` helps for the arbitrary String piece, but in order to get to 100%
prepared-statement coverage, we still need to handle the `IN` case. For that,
we can leverage the Postgres-specific [`col = ANY(ARRAY[])` syntax](https://www.postgresql.org/docs/current/functions-comparisons.html#FUNCTIONS-COMPARISONS-ANY-SOME),
turning a multi-binds `col IN (?, ?, ...)` into a single-bind `col = ANY(?)`.
There's an [open PR from 2023](https://github.com/rails/rails/pull/49388) that
does just that, but it appears stuck.

------

That's all I had. Now that I've taken the time to research and write all that,
I may as well turn it into a pull request. Stay tuned!
