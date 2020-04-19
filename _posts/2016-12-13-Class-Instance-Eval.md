---
layout: post
title: instance_eval 和 class_eval
date: 2016-12-13
tags: ruby
---

### **instance_eval** 打开对象，**class_eval** 打开类。

如果你对 Ruby 的类和对象的概念已经有一定基础的了解，你或许马上会产生疑问：类也是一种对象，那类能不能用 instance_eval 打开呢？这两种方式到底有什么区别？什么时候该用哪一个方法？

这篇文章正是要回答这三个问题。

---

首先，在理解这两个方法之前，你可能先要了解 **block**、**作用域**、**上下文** 等概念，这里我不打算介绍太多，而是希望能最快速地介绍清楚这两个方法的概念和用法。

话不多说，先通过两个例子比较一下 instance_eval 和 class_eval：

```ruby
class MyClass
    def initialize
        @v = 1
    end
end

obj = MyClss.new

obj.instance_eval do
    self                 # => #<MyClass:0x007fa90a865f50 @v=1>
    @v                   # => 1
    def show
        "obj is showing"
    end
end

obj.show                 # => "obj is showing" 
```
在这个例子中，instance_eval 方法打开了 obj 这个对象，可以看到 self 的内容就是 obj，在 obj 这个作用域内定义的方法 show 就成为了 obj 的单件方法。

```ruby
def add_method_to(a_class)
    a_class.class_eval do
        def show
            "I'm showing"
        end
    end
end

add_method_to(String)
"abc".show               # => "I'm showing" 
```
在这个例子中，我们先定义了一个 add_method_to 方法接受一个类名作为参数，然后用 class_eval 打开了这个类，本例中即 String 类，为 String 类定义了一个方法 show。通过结果可以发现，这个 show 是 String 类的实例方法，因为它能够被 String 类的一个对象 "abc" 字符串调用。

通过这两个例子，我们发现了这两个方法最大的一个区别：在 instance_eval 中定义的方法是 **单件方法**，在 class_eval 中定义的方法是 **实例方法**。所以，如果你用 instance_eval 打开一个类定义的方法就是类方法（不推荐，请往下看）。

### 以上简单的讲解可能已经满足你最初对于这两个方法的好奇心了，下面将更深入地理解这两个方法。

instance_eval 是 BasicObject 类的实例方法，它有一个孪生方法 instance_exec 可以额外接受一个 block 作为参数。
class_eval 是 Module 类的实例方法，它有一个别名 module_eval; 同样也有一个孪生方法 class_exec 可以额外接受一个 block 作为参数，这个孪生方法也有一个别名 module_exec。

本质上讲，instance_eval 的作用是为了修改 self，而 class_eval 的作用是同时修改 self 和当前类，它其实和 class 关键字一样重新打开了一个类，不过有着比 class 关键字更灵活的使用空间，它可以接受变量的调用就像上面的例子那样（事先并不知道 a_class 是什么），而 class 关键字只能使用常量，并且 class_eval 同 instance_eval 一样是使用 **扁平作用域** 的。

### 那什么情况下使用哪个方法呢？我想你心里应该已经有思路了。

事实上你会发现，一个普通对象（不是类）是不可以调用 class_eval 方法的，你会收到 `undefined method 'class_eval'` 的报错，因为 class_eval 是 Module 的实例方法，所以它只能被类调用，而且普通对象调用 class_eval 方法是没有意义的，class_eval 的作用是打开当前类，而普通对象不是类也不存在实例方法。

对类而言，当你要定义实例方法，也就是要改变当前类的时候，使用 class_eval 方法。当你要定义类方法的时候，可以使用 instance_eval 方法，而 instance_eval 方法其实是打开单件类定义单件方法的第四种方法（前三种请看 *"单件类、单件方法 和 class << obj"* ）。
class_eval 会打开当前类，而 instance_eval 会打开当前类的单件类，然而这并不是 instance_eval 的本意，它的标准含义是修改 self。