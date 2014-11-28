arel-advanced
=============

Table of contents:

1. What is Arel
2. Arel vs ActiveRecord
3. Tables, Columns
4. Terminal methods
5. Select, Where, Join, Join association, Order
6. And, Or, Less / Greater than, Not equals, etc
7. Match, In
8. Query builders

## What is Arel

Arel is a SQL AST manager for Ruby.

Let's start from a simple typical query.
```ruby
Posts.
  joins(
    'JOIN comments ON comments.post_id = posts.id',
    'JOIN authors ON authors.id = comments.author_id').
  where('authors.name = ? AND posts.active = ?', 'JDoe', true)
```

This query has some problems and confusions.
```ruby
# no syntax checking
# have to know mysql syntax
# have to write 'join' and 'on'
joins('JOIN authors ON authors.id = comments.author_id')

# no syntax checking
# not object-oriented
# have to know mysql syntax
# confusing to match arguments with question marks
where('authors.name = ? AND posts.active = ?', 'JDoe', true)
```

First of all, change joins to the symbol literals.
```ruby
Posts.
  joins(:comments).
  joins(comments: :author).
  where('authors.name = ? AND posts.active = ?', 'JDoe', true)
```

Better way: keep calm & avoid literal strings in your queries.
```ruby
Post.
  joins(:comments).
  joins(Comment.joins(:author).join_sources).
  where( Author[:name].eq('JDoe').and(Post[:active].eq(true)) )
```

What benefits ?

- no question marks
- ruby syntax checking
- object-oriented (chainable)
- don't have to know sql syntax
- easy to read - it's just ruby

## Arel vs ActiveRecord

Arel

- relational algebra for ruby
- builds SQL queries, generates AST's
- enables chaining
- applies query optimizations
- doesn't retrieve or store data
- knows nothing about your models
- knows very little about your database

ActiveRecord

- abstraction
  - no need to speak SQL dialect
- persistence
  - database rows as ruby objects
- domain logic
  - models define associations
  - models contain app logic, validations, etc

Arel constructs queries.
ActiveRecord does everything else.

## Tables, Columns

Install gem arel-helpers
```ruby
gem install arel-helpers
gem 'arel-helpers', '~> 2.0.0'
```

Include it to your models
```ruby
class Post < ActiveRecord::Base
  include ArelHelpers::ArelTable
end

Post[:id]   # Post.arel_table[:id]
Post[:text] # Post.arel_table[:text]
=> #<struct Arel::Attributes::Attribute ... >
```

## Terminal methods

ActiveRecord::Relation
```ruby
query = Post.select(:id)
query = query.select(:title)
query.to_sql
=> SELECT id, title FROM `posts`
```

The serendipity of 'select'
```ruby
Post.select(:id).to_sql
=> SELECT id FROM `posts`

Post.select(:id).count.to_sql
=> NoMethodError: undefined method 'to_sql' for 26:Fixnum
```

What happened? .count is a terminal method
```ruby
Post.select([Post[:id].count, :text]).to_sql
=> SELECT COUNT(`posts`.`id`), text FROM `posts`
```

Terminal methods

- execute immediately
- don't return an ActiveRecord::Relation
- count, first, last, to_a, pluck, each, map, etc

```ruby
# both execute the query immediately
Post.where(title: 'Arel').each_slice(3)
Post.where(title: 'Arel').each { |post| puts post.text }
```

## Select, Where, Join, Join association, Order

## And, Or, Less / Greater than, Not equals, etc

## Match, In

## Query builders
