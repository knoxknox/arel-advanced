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
https://github.com/rails/arel
http://www.rubydoc.info/github/rails/arel/Arel/Nodes/SqlLiteral
https://github.com/activerecord-hackery/squeel
http://patshaughnessy.net/2014/9/23/how-arel-converts-ruby-queries-into-sql-statements
