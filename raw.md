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
