```ruby
# Arel::SelectManager
query = Post.where(id: 100)
query = query.arel.where(Post[:id].eq(101))
query.to_sql

=> SELECT `posts`.* FROM `posts` WHERE `posts`.`id` = 100 AND `posts`.`id` = 101

# Inner Join
beer = Beer.arel_table
style = Style.arel_table
inner_join = beer.create_join(style,
  beer.create_on( beer[:style_id].eq(style[:id] ))

Beer.joins(inner_join)
=> SELECT `beers`.* FROM `beers` INNER JOIN `styles` ON `beers`.`style_id` = `styles_id`

# Outer Join
beer = Beer.arel_table
style = Style.arel_tab
outer_join = beer.create_join(style,
  beer.create_on( beer[:style_id].eq(style[:id] ), Arel::OuterJoin)

Beer.joins(outer_join)
=> SELECT `beers`.* FROM `beers` LEFT OUTER JOIN `styles` ON `beers`.`style_id` = `styles_id`

# Named functions
user = User.arel_table
left = Arel::Nodes::NamedFunction.new('LEFT', ['value', 1])
ucase = Arel::Nodes::NamedFunction.new('UCASE', [left], 'alias')
=> 'UCASE(LEFT('value', 1)) AS alias'

User.order(ucase).to_sql
=> SELECT `users`.* FROM `users` ORDER BY UCASE(LEFT('value', 1)) AS alias

User.select(ucase).to_sql
=> SELECT UCASE(LEFT('value', 1)) AS alias FROM `users`

# Aliases
user = User.arel_table
User.from(user.create_table_alias(:users, 'u')).to_sql
=> SELECT `users`.* FROM 'users' `u`

Arel::Table.new(:users, as: :u).project(Arel.star).to_sql
=> SELECT * FROM `users` `u`
```
```ruby
User.scoped.arel.
  project('UCASE(name)', Arel::SqlLiteral.new('2+2').as('num')).
  from(Arel::Table.new(:users_table, as: 'u')).as('subquery').to_sql
=> "(SELECT `users`.*, UCASE(name), 2+2 AS num FROM `users_table` `u` ) subquery"
```
SCUTTLE www.scuttle.io

```ruby
binds = ['x', 'y', 'z'].map do |v|
  Arel::Nodes::SqlLiteral.new(ActiveRecord::Base.quote_value("%#{v}%"))
end

arel_table = Arel::Table.new(:users)
User.where(arel_table[:name].matches_any(binds))
```
http://www.rubydoc.info/github/rails/arel/Arel/Nodes/SqlLiteral
https://github.com/activerecord-hackery/squeel
