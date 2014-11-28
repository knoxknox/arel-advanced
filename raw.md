## 4) SELECT, WHERE, JOIN, JOIN ASSOCIATION, ORDER

SELECT

```ruby
Post.select(Post[:visitors].sum).to_sql
=> SELECT SUM(`posts`.`visitors`) AS sum_id FROM `posts`
```

```ruby
Post.select(Post[:visitors].sum.as("visitor_total")).to_sql
=> SELECT SUM(`posts`.`views`) AS visitor_total FROM `posts`
```

```ruby
Post.select(Post[:visitors].maximum).to_sql
=> SELECT MAX(`posts`.`visitors`) AS max_id FROM `posts`
```

```ruby
Post.select(Post[:visitors].minimum).to_sql
=> SELECT MIN(`posts`.`visitors`) AS min_id FROM `posts`
```

```ruby
Post.select( Arel::Nodes::NamedFunction.new( "LENGTH", [Post[:text]] ).as("length") ).to_sql
=> SELECT LENGTH(`posts`.`text`) AS length FROM `posts`
```

```ruby
Post.select(Arel.star).to_sql
=> SELECT * FROM `posts`
```

```ruby
Post.select(:id).from(Post.select([:id, :text]).ast).to_sql
=> SELECT id FROM SELECT id, text FROM `posts`
```

WHERE

```ruby
Post.where(title: "Arel is Cool").to_sql
  => SELECT `users`.* FROM `users` WHERE `users`.`title` = 'Arel is Cool'
```

```ruby
Post.where(Post[:title].eq("Arel is Cool")).to_sql
  => SELECT `users`.* FROM `users` WHERE `users`.`title` = 'Arel is Cool' WITH AREL
```

```ruby
Post.where(Post[:title].not_eq("Arel is Cool")).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` != 'Arel is Cool')
```

```ruby
Post.where(Post[:title].not_eq(nil)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` IS NOT NULL)
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
Post.where( Post[:title].eq("Arel is Cool").and( Post[:id].eq(22).or( Post[:id].eq(23) ) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND (`posts`.`id` = 22 OR `posts`.`id` = 23) )
```

```ruby
Post.where( Post[:title].eq("Arel is Cool").and( Post[:id].in(22, 23) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND `posts`.`id` IN (22, 23) )
```

```ruby
Post.where( Post[:title].eq("Arel is Cool").and( NamedFunction.new("LENGTH", [Post[:slug]]).gt(10) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND LENGTH(`posts`.`slug`) > 10 )
```

JOIN

```ruby
class Post < ActiveRecord::Base has_many :comments end
class Author < ActiveRecord::Base belongs_to :comment end
class Comment < ActiveRecord::Base belongs_to :post has_one :author end
```

```ruby
Author.joins(:comment).where(id: 42).to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` WHERE `authors`.`id` = 42
```

```ruby
Author .joins(:comment, :comment => :post) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
Author .joins(:comment) .joins(Comment.joins(:post).join_sources) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
Author .joins( Author.arel_table.join(Comment.arel_table) .on(Comment[:id].eq(Author[:comment_id])) .join_sources ) .joins( Comment.arel_table.join(Post.arel_table) .on(Post[:id].eq(Comment[:post_id])) .join_sources ) .where(Post[:id].eq(42)) .to_sql
 => SQL
```

```ruby
Author .joins( Author.arel_table.join(Comment.arel_table, Arel::OuterJoin) .on(Comment[:id].eq(Author[:comment_id])) .join_sources ) .joins( Comment.arel_table.join(Post.arel_table, Arel::OuterJoin) .on(Post[:id].eq(Comment[:post_id])) .join_sources ) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` LEFT OUTER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

JOIN ASSOCIATION

```ruby
include ArelHelpers::JoinAssociation
Author .joins(join_association(Author, :comment, Arel::OuterJoin)) .joins(join_association(Comment, :post, Arel::OuterJoin)) .where(Post[:id].eq(42)) .to_sql
```

```ruby
include ArelHelpers::JoinAssociation
Author .joins( join_association(Author, :comment) do |assoc_name, join_conds| join_conds.and(Comment[:created_at].lteq(Date.yesterday)) end ) .joins(join_association(Comment, :post, Arel::OuterJoin)) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` AND `comments`.`created_at` <= '2014-04-15' LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
```

```ruby
class Course < ActiveRecord::Base has_and_belongs_to_many :teachers end
class Teacher < ActiveRecord::Base has_and_belongs_to_many :courses end
Course.arel_table
Teacher.arel_table
ct = Arel::Table.new(:courses_teachers)
Course .joins( Course.arel_table.join(Teacher.arel_table) .on(Course[:id].eq(ct[:course_id])) .and(Teacher[:id].eq(ct[:teacher_id])) .join_sources ) .to_sql
```

ORDER

```ruby
Post.order(:visitors).to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors
```

```ruby
Post.order(:views).reverse_order.to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors DESC
```

```ruby
Post.order(Post[:views].desc).to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors DESC
```

## 5) AND, OR , GREATER/LESS THAN, NOT EQUALS, ETC

ToDo

## 6) MATCH, IN

```ruby
Post.where( Post.arel_table[:title].in( Post.select(:title).where(id: 5).ast ) ).to_sql
  => SELECT `phrases`.* FROM `phrases` WHERE `phrases`.`title` IN ( SELECT title FROM `phrases` WHERE `phrases`.`id` = 5 )
```

```ruby
Post.where(Post[:title].matches("%arel%")).to_sql
  => SELECT `phrases`.* FROM `phrases` WHERE (`phrases`.`key` LIKE x'256172656c25')
```

## 7) QUERY BUILDERS

```ruby
class QueryBuilder extend Forwardable attr_reader :query def_delegators :@query, :to_a, :to_sql, :each def initialize(query) @query = query end protected def reflect(query) self.class.new(query) end end
```

```ruby
class PostQueryBuilder < QueryBuilder def initialize(query = nil) super(query || post.unscoped) end def with_title_matching(title) reflect( query.where(post[:title].matches("%#{title}%")) ) end def with_comments_by(usernames) reflect( query .joins(:comments => :author) .where(Author[:username].in(usernames)) ) end def since_yesterday reflect( query.where(post[:created_at].gteq(Date.yesterday)) ) end private def author Author end def post Post end end
```

```ruby
PostQueryBuilder.new .with_comments_by(['camertron', 'catwithtail']) .with_title_matching(`arel`).since_yesterday.to_sql
  => SELECT `posts`.* FROM `posts` INNER JOIN `comments` ON `comments`.`post_id` = `posts`.`id` INNER JOIN `authors` ON `authors`.`comment_id` = `comments`.`id` WHERE `authors`.`username` IN ( 'camertron', 'catwithtail' ) AND (`posts`.`title` LIKE '%arel%') AND (`posts`.`created_at` >= '2014-04-20')
```

---
SCUTTLE www.scuttle.io

```ruby
binds = ['x', 'y', 'z'].map do |v|
  Arel::Nodes::SqlLiteral.new(ActiveRecord::Base.quote_value("%#{v}%"))
end

arel_table = Arel::Table.new(:users)
User.where(arel_table[:name].matches_any(binds))
```
Part2: Hierarchy :47, AST: 48
http://patshaughnessy.net/2014/9/23/how-arel-converts-ruby-queries-into-sql-statements
