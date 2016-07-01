---
layout: post
title: rails中快速插入简单总结
date:   2016-07-01 10:23:06
categories: rails
image: /assets/images/post.jpg
---

### rails中快速插入简单总结

> ActiveRecord makes interface to the DB very easy, but it doesn't necessarily make it fast.

### Option 1: Use transactions

Instead of

```ruby
1000.times { Model.create(options) }
```

you want:

```ruby
ActiveRecord::Base.transaction do
  1000.times { Model.create(options) }
end
```

这样会由1000的事务commit请求变为一次事务commit请求。能节约一些时间(但不是很多)

### Options 2: Get down and dirty with the raw SQL

直接写sql语句

```ruby
1000.times do |i|
  Foo.connection.execute "INSERT INTO foos (counter) values (#{i})"
end
```

和上面多个请求一次事务提交一样的效果
```ruby
Foo.transaction do
  1000.times do |i|
    Foo.connection.execute "INSERT INTO foos (counter) values (#{i})"
  end
end
```

### Option 3: A single mass insert

```ruby
inserts = []
TIMES.times do
  inserts.push "(3.0, '2009-01-23 20:21:13', 2, 1)"
end
sql = "INSERT INTO user_node_scores (`score`, `updated_at`, `node_id`, `user_id`) VALUES #{inserts.join(", ")}"
```

```ruby
gem 'bulk_insert'

Book.bulk_insert(:title, :author) do |worker|
  # specify a row as an array of values...
  worker.add ["Eye of the World", "Robert Jordan"]

  # or as a hash
  worker.add title: "Lord of Light", author: "Roger Zelazny"
end
```

详细内容可以看 [bulk_insert的文档](https://github.com/jamis/bulk_insert)

这里不需要 transaction block。mass insert 已经是一个单独的事务操作。
用数组来保存字符串值的集合，然后在insert的时候，用join方法转变为符合规范的字符串。
不使用字符串的拼接。因为会产生过多的字符串生成的垃圾(拼接过程中),这样可能会引发GC,
而使性能变慢。

### Option 4: ActiveRecord::Extensions

```ruby
gem 'activerecord-import'

columns = [:score, :node_id, :user_id]
values = []
TIMES.times do
  values.push [3, 2, 1]
end

UserNodeScore.import columns, values
```

### Benchmarks

写一个简单的脚本代码

```ruby
CONN = Activerecord::Base.connection

TIMES = 10000

def do_create
  TIMES.times { User.create(name:'good name', nickname:'good nickname', age: 20, sex: 1, password_digest: 'password', email:'example@example.com')}
end

def raw_sql
  TIMES.times { CONN.execute "INSERT INTO `users` (`name`, `nickname`, `age`, `sex`, `password_digest`, `email`, `created_at`, `updated_at`) VALUES('good name', 'good nickname',20,1,'password','example@example.com','2016-07-01 11:21:13', '2016-07-01 11:21:13')" }
end

def mass_insert
  inserts = []
  TIMES.times do
    inserts.push "('good name','good nickname',20,1,'password','example@example.com','2016-07-01 11:21:13', '2016-07-01 11:21:13')"
    sql = "INSERT INTO users (`name`, `nickname`, `age`, `sex`, `password_digest`, `email`, `created_at`, `updated_at`) VALUES #{inserts.join(", ")}"
    CONN.execute sql
  end
end

def activerecord_import
  columns = [:name, :nickname, :age, :sex, :password_digest, :email, :created_at, :updated_at]
  values = []
  TIMES.times do
    values.push ['good name', 'good nickname',20,1,'password','example@example.com','2016-07-01 11:21:13', '2016-07-01 11:21:13']
  end
  User.import columns, values
end

puts "Testing various insert methods for #{TIMES} inserts\n"
puts "ActiveRecord without transaction:"
puts base = Benchmark.measure { do_create }

puts "ActiveRecord with transaction:"
puts bench = Benchmark.measure { ActiveRecord::Base.transaction { do_create } }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)

puts "Raw SQL without transaction:"
puts bench = Benchmark.measure { raw_sql }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)

puts "Raw SQL with transaction:"
puts bench = Benchmark.measure { ActiveRecord::Base.transaction { raw_sql } }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)

puts "Single mass insert:"
puts bench = Benchmark.measure { mass_insert }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)

puts "ActiveRecord::Extensions mass insert:"
puts bench = Benchmark.measure { activerecord_import }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)

puts "ActiveRecord::Extensions mass insert without validations:"
puts bench = Benchmark.measure { activerecord_import(true)  }
puts sprintf("  %2.2fx faster than base", base.real / bench.real)
```



















