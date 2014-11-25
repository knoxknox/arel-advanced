arel-advanced
=============

Query:
Posts .joins( "JOIN comments ON comments.post_id = posts.id", "JOIN authors ON authors.id = comments.author_id") .where( "authors.name = ? AND posts.active = ?", "Barack Obama", true )

Problems:
joins( "JOIN comments ON comments.post_id = posts.id", "JOIN authors ON authors.id = comments.author_id")
  HAVE TO WRITE “JOIN” AND “ON”
  HAVE TO KNOW MYSQL SYNTAX
  NO SYNTAX CHECKING
where( "authors.name = ? AND posts.active = ?", "Barack Obama", true )
  HAVE TO KNOW MYSQL SYNTAX
  CONFUSING TO MATCH ARGUMENTS WITH QUESTION MARKS
  NOT OBJECT-ORIENTED
  NO SYNTAX CHECKING

Rewrite query:
Posts .joins(:comments) .joins(:comments => :author) .where( "authors.name = ? AND posts.active = ?", "Barack Obama", true )

KEEP CALM AND AVOID LITERAL STRINGS IN YOUR QUERIES

A BETTER WAY
Post .joins(:comments) .joins(Comment.joins(:author).join_sources) .where( Author[:name].eq("Barack Obama") .and(Post[:active].eq(true)) )
  DON’T HAVE TO KNOW SQL SYNTAX
  RUBY SYNTAX CHECKING
  OBJECT-ORIENTED (CHAINABLE)
  NO QUESTION MARKS
  EASY TO READ - IT’S JUST RUBY!

1) ACTIVERECORD VS AREL

ACTIVERECORD
  DATABASE ABSTRACTION
    NO NEED TO SPEAK A DIALECT OF SQL
  PERSISTENCE
    DATABASE ROWS AS RUBY OBJECTS
  DOMAIN LOGIC
    MODELS CONTAIN APPLICATION LOGIC, VALIDATIONS, ETC
    MODELS DEFINE ASSOCIATIONS

AREL
  “RELATIONAL ALGEBRA” FOR RUBY
  BUILDS SQL QUERIES, GENERATES ASTS
  APPLIES QUERY OPTIMIZATIONS
  ENABLES CHAINING
  “VEXINGLY UNDOCUMENTED”
  KNOWS NOTHING ABOUT YOUR MODELS
  KNOWS VERY LITTLE ABOUT YOUR DATABASE
  DOES NOT RETRIEVE OR STORE DATA

AREL CONSTRUCTS QUERIES
ACTIVERECORD DOES EVERYTHING ELSE

2) TABLES, COLUMNS

AREL-HELPERS GEM
gem install arel-helpers
gem “arel-helpers”, “~> 1.1.0”

class Post < ActiveRecord::Base has_many :comments end
Post.arel_table[:id]
Post.arel_table[:text]
=> #<struct Arel::Attributes::Attribute ... >

class Post < ActiveRecord::Base include ArelHelpers::ArelTable has_many :comments end
Post[:id] # Post.arel_table[:id]
Post[:text] # Post.arel_table[:text]
=> #<struct Arel::Attributes::Attribute ... >

RELATIONS
query = Post.select(:title)
query = query.select(:id)
query.to_sql
  => SELECT title, id FROM `posts`

3) TERMINAL METHODS

Post.select([:id, :text]).to_sql
  => SELECT id, text FROM `posts`
Post.select(:id).count.to_sql
  => NoMethodError: undefined method `to_sql' for 26:Fixnum
WHAT HAPPENED?? .count IS A TERMINAL METHOD

Post.select([Post[:id].count, :text]).to_sql
  => SELECT COUNT(`posts`.`id`), text FROM `posts`

TERMINAL METHODS
  EXECUTE IMMEDIATELY
  DO NOT RETURN AN ActiveRecord::Relation (count first, last to_a pluck each, map, ETC)

BOTH EXECUTE THE QUERY IMMEDIATELY
  Post.where(title: "Arel is Cool").each do |post| puts post.text end
  Post.where(title: "Arel is Cool").each_slice(3)

4) SELECT, WHERE, JOIN, JOIN ASSOCIATION, ORDER

Post.select(Post[:visitors].sum).to_sql
  => SELECT SUM(`posts`.`visitors`) AS sum_id FROM `posts`
Post.select(Post[:visitors].sum.as("visitor_total")).to_sql
  => SELECT SUM(`posts`.`views`) AS visitor_total FROM `posts`
Post.select(Post[:visitors].maximum).to_sql
  => SELECT MAX(`posts`.`visitors`) AS max_id FROM `posts`
Post.select(Post[:visitors].minimum).to_sql
  => SELECT MIN(`posts`.`visitors`) AS min_id FROM `posts`
Post.select( Arel::Nodes::NamedFunction.new( "LENGTH", [Post[:text]] ).as("length") ).to_sql
  => SELECT LENGTH(`posts`.`text`) AS length FROM `posts`
Post.select(Arel.star).to_sql
  => SELECT * FROM `posts`
Post.select(:id).from(Post.select([:id, :text]).ast).to_sql
  => SELECT id FROM SELECT id, text FROM `posts`

Post.where(title: "Arel is Cool").to_sql
  => SELECT `users`.* FROM `users` WHERE `users`.`title` = 'Arel is Cool'
Post.where(Post[:title].eq("Arel is Cool")).to_sql
  => SELECT `users`.* FROM `users` WHERE `users`.`title` = 'Arel is Cool' WITH AREL
Post.where(Post[:title].not_eq("Arel is Cool")).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` != 'Arel is Cool')
Post.where(Post[:title].not_eq(nil)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`title` IS NOT NULL)
Post.where(Post[:visitors].gt(250)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` > 250)
Post.where(Post[:visitors].lt(250)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` < 250)
Post.where(Post[:visitors].gteq(250)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` >= 250)
Post.where(Post[:visitors].lteq(250)).to_sql
  => SELECT `posts`.* FROM `posts` WHERE (`posts`.`visitors` <= 250)
Post.where( Post[:title].eq("Arel is Cool").and( Post[:id].eq(22).or( Post[:id].eq(23) ) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND (`posts`.`id` = 22 OR `posts`.`id` = 23) )
Post.where( Post[:title].eq("Arel is Cool").and( Post[:id].in(22, 23) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND `posts`.`id` IN (22, 23) )
Post.where( Post[:title].eq("Arel is Cool").and( NamedFunction.new("LENGTH", [Post[:slug]]).gt(10) ) ).to_sql
  => SELECT `posts`.* FROM `posts` WHERE ( `posts`.`title` = 'Arel is Cool' AND LENGTH(`posts`.`slug`) > 10 )

class Post < ActiveRecord::Base has_many :comments end
class Author < ActiveRecord::Base belongs_to :comment end
class Comment < ActiveRecord::Base belongs_to :post has_one :author end

Author.joins(:comment).where(id: 42).to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` WHERE `authors`.`id` = 42
Author .joins(:comment, :comment => :post) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
Author .joins(:comment) .joins(Comment.joins(:post).join_sources) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` INNER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42
Author .joins( Author.arel_table.join(Comment.arel_table) .on(Comment[:id].eq(Author[:comment_id])) .join_sources ) .joins( Comment.arel_table.join(Post.arel_table) .on(Post[:id].eq(Comment[:post_id])) .join_sources ) .where(Post[:id].eq(42)) .to_sql
 => SQL
Author .joins( Author.arel_table.join(Comment.arel_table, Arel::OuterJoin) .on(Comment[:id].eq(Author[:comment_id])) .join_sources ) .joins( Comment.arel_table.join(Post.arel_table, Arel::OuterJoin) .on(Post[:id].eq(Comment[:post_id])) .join_sources ) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` LEFT OUTER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42

include ArelHelpers::JoinAssociation
Author .joins(join_association(Author, :comment, Arel::OuterJoin)) .joins(join_association(Comment, :post, Arel::OuterJoin)) .where(Post[:id].eq(42)) .to_sql

include ArelHelpers::JoinAssociation
Author .joins( join_association(Author, :comment) do |assoc_name, join_conds| join_conds.and(Comment[:created_at].lteq(Date.yesterday)) end ) .joins(join_association(Comment, :post, Arel::OuterJoin)) .where(Post[:id].eq(42)) .to_sql
  => SELECT `authors`.* FROM `authors` INNER JOIN `comments` ON `comments`.`id` = `authors`.`comment_id` AND `comments`.`created_at` <= '2014-04-15' LEFT OUTER JOIN `posts` ON `posts`.`id` = `comments`.`post_id` WHERE `posts`.`id` = 42

class Course < ActiveRecord::Base has_and_belongs_to_many :teachers end
class Teacher < ActiveRecord::Base has_and_belongs_to_many :courses end
Course.arel_table
Teacher.arel_table
ct = Arel::Table.new(:courses_teachers)
Course .joins( Course.arel_table.join(Teacher.arel_table) .on(Course[:id].eq(ct[:course_id])) .and(Teacher[:id].eq(ct[:teacher_id])) .join_sources ) .to_sql

Post.order(:visitors).to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors
Post.order(:views).reverse_order.to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors DESC
Post.order(Post[:views].desc).to_sql
  => SELECT `posts`.* FROM `posts` ORDER BY visitors DESC

5) AND, OR , GREATER/LESS THAN, NOT EQUALS, ETC
6) MATCH, IN

Post.where( Post.arel_table[:title].in( Post.select(:title).where(id: 5).ast ) ).to_sql
  => SELECT `phrases`.* FROM `phrases` WHERE `phrases`.`title` IN ( SELECT title FROM `phrases` WHERE `phrases`.`id` = 5 )

Post.where(Post[:title].matches("%arel%")).to_sql
  => SELECT `phrases`.* FROM `phrases` WHERE (`phrases`.`key` LIKE x'256172656c25')

7) QUERY BUILDERS

class QueryBuilder extend Forwardable attr_reader :query def_delegators :@query, :to_a, :to_sql, :each def initialize(query) @query = query end protected def reflect(query) self.class.new(query) end end

class PostQueryBuilder < QueryBuilder def initialize(query = nil) super(query || post.unscoped) end def with_title_matching(title) reflect( query.where(post[:title].matches("%#{title}%")) ) end def with_comments_by(usernames) reflect( query .joins(:comments => :author) .where(Author[:username].in(usernames)) ) end def since_yesterday reflect( query.where(post[:created_at].gteq(Date.yesterday)) ) end private def author Author end def post Post end end

PostQueryBuilder.new .with_comments_by(['camertron', 'catwithtail']) .with_title_matching(`arel`).since_yesterday.to_sql
  => SELECT `posts`.* FROM `posts` INNER JOIN `comments` ON `comments`.`post_id` = `posts`.`id` INNER JOIN `authors` ON `authors`.`comment_id` = `comments`.`id` WHERE `authors`.`username` IN ( 'camertron', 'catwithtail' ) AND (`posts`.`title` LIKE '%arel%') AND (`posts`.`created_at` >= '2014-04-20')

SCUTTLE www.scuttle.io
