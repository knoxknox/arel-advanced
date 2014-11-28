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

## Tables, Columns

## Terminal methods

## Select, Where, Join, Join association, Order

## And, Or, Less / Greater than, Not equals, etc

## Match, In

## Query builders
