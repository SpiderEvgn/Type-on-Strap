---
layout: post
title: Includes vs Joins
date: 2017-03-21
tags: rails
---

### 在说 Rails 的 includes 和 joins 之前，先来回顾一下 SQL 中的 inner join 和 left join 分别是什么，稍后会用到。

**inner join:** 只返回两个表中联结字段相等的行        
**left join:**  返回包括左表中的所有记录和右表中联结字段相等的记录 

举一个小例子：

```
表A记录如下：
aID　　　　　aNum
1　　　　　  a1
2　　　　　  a2
3　　　　　  a3
4　　　　　  a4
5　　　　　  a5

表B记录如下:
bID　　　　　bName
1　　　　　  b1
2　　　　　  b2
3　　　　　  b3
4　　　　　  b4
8　　　　　  b8
--------------------------------------------
1. inner join
sql语句如下: 
select * from A inner join B on A.aID = B.bID

结果如下:
aID　　　　　aNum　　　　　bID　　　　　bName
1　　　　　  a1　　　　    1　　　　　  b1
2　　　　　  a2　　　　    2　　　　　  b2
3　　　　　  a3　　　　    3　　　　　  b3
4　　　　　  a4　　　　    4　　　　　  b4

结果说明:
很明显，这里只显示出了 A.aID = B.bID 的记录，这说明 inner join 只显示符合条件的记录。
--------------------------------------------
2. left join
sql语句如下: 
select * from A left join B on A.aID = B.bID

结果如下:
aID　　　　　aNum　　　　　bID　　　　　bName
1　　　　　  a1　　　　    1　　　　　  b1
2　　　　　  a2　　　　    2　　　　　  b2
3　　　　　  a3　　　　    3　　　　　  b3
4　　　　　  a4　　　　    4　　　　　  b4
5　　　　　  a5　　　　    NULL　　　　 NULL

结果说明:
left join 是以 A 表的记录为基础的, A 可以看成左表, B 可以看成右表, left join 是以左表为准的.
换句话说,左表(A)的记录将会全部表示出来, 而右表(B)只会显示符合搜索条件的记录(例子中为: A.aID = B.bID).
B表记录不足的地方均为NULL.
--------------------------------------------
```

*注：left join 是 left outer join 的缩写* 

### 下面从一个小例子来理解 includes 和 joins 的区别

以下是 Group 和 Post 的表内容（ `Group has_many posts` ），为了方便阅读我调整了格式：

```ruby
2.3.0 :100 > Group.all
  Group Load (4.9ms)  SELECT "groups".* FROM "groups"
 => #<ActiveRecord::Relation [
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">, 
 #<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">, 
 #<Group id: 3, title: "333333", description: "this is group 3", created_at: "2016-12-21 12:35:43", updated_at: "2016-12-21 12:35:43">
 ]> 
2.3.0 :101 > Post.all
  Post Load (2.7ms)  SELECT "posts".* FROM "posts"
 => #<ActiveRecord::Relation [
 #<Post id: 1, content: "group_1_111111", group_id: 1, created_at: "2016-12-21 12:36:45", updated_at: "2016-12-21 12:36:45">, 
 #<Post id: 2, content: "group_1_222222", group_id: 1, created_at: "2016-12-21 12:36:51", updated_at: "2016-12-21 12:36:51">, 
 #<Post id: 3, content: "group_1_333333", group_id: 1, created_at: "2016-12-21 12:36:55", updated_at: "2016-12-21 12:36:55">, 
 #<Post id: 4, content: "group_2_111111", group_id: 2, created_at: "2016-12-21 12:37:05", updated_at: "2016-12-21 12:37:05">, 
 #<Post id: 5, content: "group_2_222222", group_id: 2, created_at: "2016-12-21 12:37:09", updated_at: "2016-12-21 12:37:09">, 
 #<Post id: 6, content: "group_1_444444", group_id: 1, created_at: "2016-12-21 12:37:17", updated_at: "2016-12-21 12:37:17">
 ]> 
```
先来看看直接用 joins 和 includes 的结果有什么不同：

```ruby
2.3.0 :111 >   Group.joins(:posts)
  Group Load (2.2ms)  SELECT "groups".* FROM "groups" INNER JOIN "posts" ON "posts"."group_id" = "groups"."id"
 => #<ActiveRecord::Relation [
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">, 
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">, 
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">, 
 #<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">, 
 #<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">, 
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">]> 
2.3.0 :112 > 
2.3.0 :113 >   Group.includes(:posts)
  Group Load (0.9ms)  SELECT "groups".* FROM "groups"
  Post Load (0.5ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" IN (1, 2, 3)
 => #<ActiveRecord::Relation [
 #<Group id: 1, title: "111111", description: "this is group 1", created_at: "2016-12-21 12:35:25", updated_at: "2016-12-21 12:35:25">, 
 #<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">, 
 #<Group id: 3, title: "333333", description: "this is group 3", created_at: "2016-12-21 12:35:43", updated_at: "2016-12-21 12:35:43">]> 
2.3.0 :114 > 
```
joins 只请求了一次数据库查询，只取回 groups 表的数据，它会有重复记录，因为是通过 inner join 的方式。
includes 请求了两次数据库查询，取回了 groups 的所有数据和 posts 中与 groups 关联的所有数据。其实这就是 left join 的方式，在后面结合 where 子句使用的情况下会更清晰的看到。

来看下面这段代码的输出：

```ruby
ro = Group.all
ro.each do |r|
  r.posts.each do |p|
    puts p.content
  end
end
--------------------------------------------
  Group Load (0.8ms)  SELECT "groups".* FROM "groups"
  Post Load (0.7ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 1]]
group_1_111111
group_1_222222
group_1_333333
group_1_444444
  Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 2]]
group_2_111111
group_2_222222
  Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 3]]
```
这个就是很著名的 “`N+1 query`” 问题，当我用 ro 拿到所有的 group 数据后，每一个 r.posts 都会触发一次数据库查询请求，例子中有 3 条 group 记录就产生 3 条请求，如果数据库足够大，那么这个巨大的时间成本是不可承受的。

### 解决方法就是利用 includes，因为 rails 提供的 includes 方法会 eager loading 所有的数据：

```ruby
ri = Group.includes(:posts).all
ri.each do |r|
  r.posts.each do |p|
    puts p.content
  end
end
--------------------------------------------
  Group Load (0.4ms)  SELECT "groups".* FROM "groups"
  Post Load (0.8ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" IN (1, 2, 3)
group_1_111111
group_1_222222
group_1_333333
group_1_444444
group_2_111111
group_2_222222
```
正如之前分析的，includes 的两次查询同时取回了 groups 和 posts 中的所有数据放入内存，所以在之后的 r.posts 中程序直接就从内存中获取了 post 的 content，而不用再针对每条记录做一次数据库的查询了，很强大吧。

那么 joins 在这个情况的表现如何呢？

```ruby
rj = Group.joins(:posts).all
rj.each do |r|
  r.posts.each do |p|
    puts p.content
  end
end
--------------------------------------------
  Group Load (0.9ms)  SELECT "groups".* FROM "groups" INNER JOIN "posts" ON "posts"."group_id" = "groups"."id"
  Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 1]]
group_1_111111
group_1_222222
group_1_333333
group_1_444444
  Post Load (0.4ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 1]]
group_1_111111
group_1_222222
group_1_333333
group_1_444444
  Post Load (0.9ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 1]]
group_1_111111
group_1_222222
group_1_333333
group_1_444444
  Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 2]]
group_2_111111
group_2_222222
  Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 2]]
group_2_111111
group_2_222222
  Post Load (0.3ms)  SELECT "posts".* FROM "posts" WHERE "posts"."group_id" = $1  [["group_id", 1]]
group_1_111111
group_1_222222
group_1_333333
group_1_444444
```
因为只取回 groups 表的数据，并且通过 inner join 的方式会有重复记录，所以 joins 完全不符合这个场景的需求。

### 那什么时候会用到 joins 呢？这就到了 where 出场的时候了。当我要获取 group 的数据，而筛选条件是与其关联的 post 的时候，我们来看具体代码：

```ruby
2.3.0 :120 >   Group.joins(:posts).where(posts: {content: 'group_2_111111'})
  Group Load (3.7ms)  SELECT "groups".* FROM "groups" INNER JOIN "posts" ON "posts"."group_id" = "groups"."id" WHERE "posts"."content" = $1  [["content", "group_2_111111"]]
 => #<ActiveRecord::Relation [#<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">]> 
```
显然，joins 的查询方式很好的满足了需求，并且只返回 group 的数据。那如果在 includes 上用 where 子句会怎么样呢？

```ruby
2.3.0 :121 >   Group.includes(:posts).where(posts: {content: 'group_2_111111'})
  SQL (1.6ms)  SELECT "groups"."id" AS t0_r0, "groups"."title" AS t0_r1, "groups"."description" AS t0_r2, "groups"."created_at" AS t0_r3, "groups"."updated_at" AS t0_r4, "posts"."id" AS t1_r0, "posts"."content" AS t1_r1, "posts"."group_id" AS t1_r2, "posts"."created_at" AS t1_r3, "posts"."updated_at" AS t1_r4 FROM "groups" LEFT OUTER JOIN "posts" ON "posts"."group_id" = "groups"."id" WHERE "posts"."content" = $1  [["content", "group_2_111111"]]
 => #<ActiveRecord::Relation [#<Group id: 2, title: "222222", description: "this is group 2", created_at: "2016-12-21 12:35:34", updated_at: "2016-12-21 12:35:34">]> 
```
发现数据库的查询请求不再是原来的两条，而合并成了一条同时返回 group 和 post 的数据，并且是通过 left outer join 的方式连接表。结果虽然相同，但是 includes 消耗了多余的内存，并且查询逻辑也不完全匹配。关于 includes 结合 where 的使用，建议还是仔细阅读 [rails guides](http://guides.rubyonrails.org/active_record_querying.html#joins) 的官方文档。

>最后做一个总结：                 
joins 的作用是连接两张表，一般是结合条件查询使用，当需要获取 A 表中的数据而查询条件是与 A 关联的 B 表时，使用 joins。includes 则是当需要使用 A、B 两张关联数据表的时候，用 includes 方法将数据一次 eager loading 进内存，以此减少之后大量的数据库读取操作。

* 最后的最后，提一个小细节，因为关联关系是 `group has_many posts`, 所以当 includes 和 joins 的顺序反过来的时候 groups 表要用单数。

```ruby
2.3.0 :126 >   Post.joins(:groups)
ActiveRecord::ConfigurationError: Can't join 'Post' to association named 'groups'; perhaps you misspelled it?
 .
 .
 .
2.3.0 :127 >   Post.joins(:group)
  Post Load (1.3ms)  SELECT "posts".* FROM "posts" INNER JOIN "groups" ON "groups"."id" = "posts"."group_id"
 => #<ActiveRecord::Relation [
 #<Post id: 1, content: "group_1_111111", group_id: 1, created_at: "2016-12-21 12:36:45", updated_at: "2016-12-21 12:36:45">, 
 #<Post id: 2, content: "group_1_222222", group_id: 1, created_at: "2016-12-21 12:36:51", updated_at: "2016-12-21 12:36:51">, 
 #<Post id: 3, content: "group_1_333333", group_id: 1, created_at: "2016-12-21 12:36:55", updated_at: "2016-12-21 12:36:55">, 
 #<Post id: 4, content: "group_2_111111", group_id: 2, created_at: "2016-12-21 12:37:05", updated_at: "2016-12-21 12:37:05">, 
 #<Post id: 5, content: "group_2_222222", group_id: 2, created_at: "2016-12-21 12:37:09", updated_at: "2016-12-21 12:37:09">, 
 #<Post id: 6, content: "group_1_444444", group_id: 1, created_at: "2016-12-21 12:37:17", updated_at: "2016-12-21 12:37:17">]> 
2.3.0 :128 > Post.includes(:group)
  Post Load (5.5ms)  SELECT "posts".* FROM "posts"
  Group Load (1.0ms)  SELECT "groups".* FROM "groups" WHERE "groups"."id" IN (1, 2)
 => #<ActiveRecord::Relation [
 #<Post id: 1, content: "group_1_111111", group_id: 1, created_at: "2016-12-21 12:36:45", updated_at: "2016-12-21 12:36:45">, 
 #<Post id: 2, content: "group_1_222222", group_id: 1, created_at: "2016-12-21 12:36:51", updated_at: "2016-12-21 12:36:51">, 
 #<Post id: 3, content: "group_1_333333", group_id: 1, created_at: "2016-12-21 12:36:55", updated_at: "2016-12-21 12:36:55">, 
 #<Post id: 4, content: "group_2_111111", group_id: 2, created_at: "2016-12-21 12:37:05", updated_at: "2016-12-21 12:37:05">, 
 #<Post id: 5, content: "group_2_222222", group_id: 2, created_at: "2016-12-21 12:37:09", updated_at: "2016-12-21 12:37:09">, 
 #<Post id: 6, content: "group_1_444444", group_id: 1, created_at: "2016-12-21 12:37:17", updated_at: "2016-12-21 12:37:17">]> 
```

---

## 参考资料：

* [http://guides.rubyonrails.org/active_record_querying.html#joins](http://guides.rubyonrails.org/active_record_querying.html#joins)
* [http://www.cnblogs.com/pcjim/articles/799302.html](http://www.cnblogs.com/pcjim/articles/799302.html)
* [http://railscasts.com/episodes/181-include-vs-joins?autoplay=true](http://railscasts.com/episodes/181-include-vs-joins?autoplay=true)
* [http://blog.bigbinary.com/2013/07/01/preload-vs-eager-load-vs-joins-vs-includes.html](http://blog.bigbinary.com/2013/07/01/preload-vs-eager-load-vs-joins-vs-includes.html)
* [http://tomdallimore.com/blog/includes-vs-joins-in-rails-when-and-where/](http://tomdallimore.com/blog/includes-vs-joins-in-rails-when-and-where/)
* [http://stackoverflow.com/questions/1208636/rails-include-vs-joins](http://stackoverflow.com/questions/1208636/rails-include-vs-joins)