arel-advanced
=============

Table of contents:

1. What is Arel
2. Arel vs ActiveRecord
3. Tables, Columns
4. Terminal methods
5. Select, Where, Order, Join, Join association
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

## Select, Where, Order, Join, Join association

SELECT

```ruby
Post.select(Arel.star).to_sql
=> SELECT `posts`.* FROM `posts`
```

```ruby
subquery = Post.select([:id, :text])
Post.select(:id).from(subquery.ast).to_sql
=> SELECT id FROM SELECT id, text FROM `posts`
```

```ruby
Post.select(Post[:visitors].sum).to_sql
=> SELECT SUM(`posts`.`visitors`) AS sum_id FROM `posts`
```

```ruby
Post.select(Post[:visitors].minimum).to_sql
=> SELECT MIN(`posts`.`visitors`) AS min_id FROM `posts`
```

```ruby
Post.select(Post[:visitors].maximum).to_sql
=> SELECT MAX(`posts`.`visitors`) AS max_id FROM `posts`
```

```ruby
Post.select(Post[:visitors].sum.as('visitors_total')).to_sql
=> SELECT SUM(`posts`.`visitors`) AS visitors_total FROM `posts`
```

```ruby
Post.select(Arel::Nodes::NamedFunction.new('LENGTH', [Post[:text]]).as('length')).to_sql
=> SELECT LENGTH(`posts`.`text`) AS length FROM `posts`
```

WHERE

```ruby
Post.where(title: 'Arel is Cool').to_sql
=> SELECT `posts`.* FROM `posts` WHERE `posts`.`title` = 'Arel is Cool'
```

```ruby
Post.where(Post[:visitors].gt(250)).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` > 250)
```

```ruby
Post.where(Post[:visitors].lt(250)).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` < 250)
```

```ruby
Post.where(Post[:visitors].gteq(250)).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` >= 250)
```

```ruby
Post.where(Post[:visitors].lteq(250)).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` <= 250)
```

```ruby
Post.where(Post[:title].not_eq(nil)).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` IS NOT NULL)
```

```ruby
Post.where(Post[:title].eq('Arel is Cool')).to_sql
=> SELECT `posts`.* FROM `posts` WHERE `posts`.`title` = 'Arel is Cool'
```

```ruby
Post.where(Post[:title].not_eq('Arel is Cool')).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` != 'Arel is Cool')
```

```ruby
Post.where(Post[:title].eq('Arel is Cool').and(Post[:id].in(22, 23))).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` = 'Arel is Cool' AND `posts`.`id` IN(22, 23))
```

```ruby
Post.where(Post[:title].eq('Arel is Cool').and(NamedFunction.new('LENGTH', [Post[:slug]]).gt(10))).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` = 'Arel is Cool' AND LENGTH(`posts`.`slug`) > 10)
```

```ruby
Post.where(Post[:title].eq('Arel is Cool').and(Post[:id].eq(22).or(Post[:id].eq(23)))).to_sql
=> SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` = 'Arel is Cool' AND (`posts`.`id` = 22 OR `posts`.`id` = 23))
```

ORDER

```ruby
Post.order(:views).to_sql
=> SELECT `posts`.* FROM `posts` ORDER BY views
```

```ruby
Post.order(Post[:views].desc).to_sql
=> SELECT `posts`.* FROM `posts` ORDER BY views DESC
```

```ruby
Post.order(:views).reverse_order.to_sql
=> SELECT `posts`.* FROM `posts` ORDER BY views DESC
```

JOIN

```ruby
# relations example
Post:    has_many(:comments)
Author:  belongs_to(:comment)
Comment: belongs_to(:post), has_one(:author)
```

```ruby
Author.joins(:comment).where(id: 42).to_sql
=> SELECT `authors`.* FROM `authors`
   INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id`
   WHERE `authors`.`id` = 42
```

```ruby
Author.joins(:comment, comment: :post).where(Post[:id].eq(42)).to_sql
=> SELECT `authors`.* FROM `authors`
   INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id`
   INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
Author.
  joins(:comment).
  joins(Comment.joins(:post).join_sources).where(Post[:id].eq(42)).to_sql

=> SELECT `authors`.* FROM `authors`
   INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id`
   INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
Author.
  joins(
    Author.arel_table.join(
      Comment.arel_table, Arel::OuterJoin).on(Comment[:id].eq(Author[:comment_id])
    ).join_sources
  ).
  joins(
    Comment.arel_table.join(
      Post.arel_table, Arel::OuterJoin).on(Post[:id].eq(Comment[:post_id])
    ).join_sources
  ).where(Post[:id].eq(42)).to_sql

=> SELECT `authors`.* FROM `authors`
   LEFT OUTER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id`
   LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

JOIN ASSOCIATION

```ruby
include ArelHelpers::JoinAssociation
Author.
  joins(join_association(Author, :comment, Arel::OuterJoin)).
  joins(join_association(Comment, :post, Arel::OuterJoin)).where(Post[:id].eq(42)).to_sql
```

```ruby
include ArelHelpers::JoinAssociation
Author.
  joins(join_association(Author, :comment) { |assoc_name, join_conds|
    join_conds.and(Comment[:created_at].lteq(Date.yesterday))
  }).
  joins(join_association(Comment, :post, Arel::OuterJoin)).where(Post[:id].eq(42)).to_sql

=> SELECT `authors`.* FROM `authors`
   INNER JOIN `comments` ON
     `comments`.`id` = `authors`.`comment_id`AND `comments`.`created_at` <= '2014-04-15'
   LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
# relations example
Course: has_and_belongs_to_many(:teachers)
Teacher: has_and_belongs_to_many(:courses)

Course.arel_table
Teacher.arel_table
ct = Arel::Table.new(:courses_teachers)

Course.joins(
  Course.arel_table.join(Teacher.arel_table).
    on(Course[:id].eq(ct[:course_id])).and(
      Teacher[:id].eq(ct[:teacher_id])).join_sources).to_sql
```

## And, Or, Less / Greater than, Not equals, etc

## Match, In

## Query builders
