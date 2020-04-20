---
layout: post
title: Polymorphic Association（多态关联）
date: 2017-02-12
tags: rails
---

Polymorphic Association（多态关联）这个名字听上去很高大上，好像是一个很厉害又很复杂的功能，其实不然。

多态关联这个概念非常简单，举一个简单的小例子就明白了：

```ruby
class Comment < ApplicationRecord
end

class Article < ApplicationRecord
end

class Photo < ApplicationRecord
end

```
这里我们有三个类，Article 和 Photo 都可以被留言，也就是他们分别和 Comment 是一对多的关系。在没有用多态的情况下，我们原来的做法是 Artile 和 ArticleComment（外键: article_id） 之间建立 has_many / belongs_to 的关系，Photo 再和 PhotoComment（外键: photo_id） 之间建立 has_many / belongs_to 的关系，但是两个 comment 其实结构完全相同，同样是一个评论的模型，只不过和别的多个模型之间产生关联关系。

> 当一个模型需要同时属于其他多个模型的时候，使用多态关联（polymorphic）。

那么如何做到呢？我们来看 Rails 的具体实现：

```ruby
class Comment < ApplicationRecord
    belongs_to :commentable, polymorphic: true
end

class Article < ApplicationRecord
    has_many :comments, as: :commentable
end

class Photo < ApplicationRecord
    has_many :comments, as: :commentable
end
```
通过建立一个 commentable 的虚拟接口，并且指明 polymorphic: true，来让其他多个模型建立关系从而实现多态关联。但是，comment 如何知道自己关联的对象是 article 还是 photo 呢？按原来的模式，外键应该是 article_id 或者是 photo_id，多态是如何做的呢？

秘诀就是新增一个字段: commentable_type。来看 migration:

```ruby
class CreateComments < ActiveRecord::Migration
    def change
        create_table :comments do |t|
            t.text :content
            t.integer :commentable_id
            t.string :commentable_type

            t.timestamps
        end
    end
end
```
我们用 content 来储存留言的内容，commentable_id 储存被留言的对象 id ，commentable_type 则用来储存被留言对象的种类，以这个例子来说被留言的对象就是 Article 与 Photo 这两种 Model。

看一下 console 中的实际结果：

```ruby
article = Article.first
comment_a = article.comments.create(content: "First Article Comment")

# 你可以发现 Rails 很聪明的帮我们指定了被留言对象的 type 和 id
comment_a.id                      # => 1
comment_a.commentable_type        # => "Article"
comment_a.commentable_id          # => 1

photo = Photo.first
comment_p = photo.comments.create(content: "First Photo Comment")

# 因为共用一张 comments 表，所有 comment_p 的 id 索引增加了
comment_p.id                      # => 2
comment_p.commentable_type        # => "Photo"
comment_p.commentable_id          # => 1

# 也可以通过 commentable 反向回查关联的对象
comment_a.commentable => #<Article id: 1, ....>
comment_p.commentable => #<Photo id: 1, ....>
```

**本文主要介绍 Rails 中多态关联的概念，关于路由和表单中如何使用多态，推荐看 railscasts 的相关介绍，链接在参考资料**

---

## 参考资料：

* [http://railscasts.com/episodes/154-polymorphic-association?autoplay=true](http://railscasts.com/episodes/154-polymorphic-association?autoplay=true)
* [https://ihower.tw/rails/activerecord-relationships-cn.html#sec6](https://ihower.tw/rails/activerecord-relationships-cn.html#sec6)
* [http://guides.rubyonrails.org/association_basics.html](http://guides.rubyonrails.org/association_basics.html)

