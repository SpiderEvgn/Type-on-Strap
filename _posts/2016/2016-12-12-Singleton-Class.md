---
layout: post
title: 单件类、单件方法 和 class << obj
date: 2016-12-12
tags: ruby
---

### **单件方法** 指的是只针对一个对象的方法，**单件类** 就是定义这些单件方法的作用域。

首先，讲讲如何定义 *单件方法* (singleton_methods) 和 *单件方法* 的作用。

比如我们有一个 Person 类，Person 类有一个对象 aaron, Person类还有实例方法 hear 和 see, 显然，所有 Person 的实例对象都可以调用 hear 和 see 方法，包括 aaron。但是，如果我只想让 aaron 可以 talk，就不能在 Person 类中定义实例方法，这样，所有对象就都能 talk 了。解决方案就是，定义 aaron 对象的 *单件方法*:

```ruby
class Person
    def hear
        'hear'
    end
    def see
        'see'
    end
end

aaron = Person.new

def aaron.talk
    # do something
end


aaron.class                            # => Person 
aaron.class.instance_methods(false)    # => [:see, :hear]
aaron.methods(false)                   # => [:talk]   (false 表示只显示自己的方法，不显示继承来的方法)
```
可见，talk 是只属于 aaron 对象的方法。那么，talk 到底是在哪里定义的呢？换句话说，talk 方法是哪个类的实例方法呢？接下来就谈谈 **单件类**。

---

ruby 中 **单件类**（singleton class，或者叫单例类）是个很特殊的类，像 `Object#class` 这样的方法会小心翼翼地把单件类隐藏起来。 不过 Ruby 提供了一种特殊的基于 class 关键字的语法，可以让你进入该单件类的作用域：

```ruby
class << aaron
    # your code
end
```
如果想获得这个单件类的引用，可以在离开该作用域时返回 self

```ruby
aaron = Person.new

singleton_class = class << aaron
    self
end

singleton_class                # => #<Class:#<Person:0x007fe42485bc80>>
```
用 `class <<` 方法最直接的打开了单件类的作用域，由此得到了第二种定义单件方法的方式：

```ruby
class << aaron
    def cry
        # do something
    end
end

aaron.methods(false)                   # => [:talk, :cry]  
```
此外，Ruby 还提供了一个更加方便的语法 `Object#singleton_class` 来获取单件类:

```ruby
aaron.singleton_class                           # => #<Class:#<Person:0x007fe42485bc80>> 
aaron.singleton_class.instance_methods(false)   # => [:talk, :shout] 
aaron.singleton_class.superclass                # => Person
```
可见，一个对象的单件方法其实是继承自这个对象本身的类的，所以在搜索方法时，会先“向右一步“搜索这个对象单件类中的实例方法，也就是该对象的单件方法，然后再”向上一步“依次在祖先链中搜索超类的实例方法。

### 到此，你大概对 **单件类**、**单件方法** 和 **`class << obj`** 有一个感性的认识了。
最后，讲一讲类的单件类和单件方法。

因为 ruby 中的类其实也是对象，所以类也有单件类和单件方法。你可能已经发现了，其实类的单件方法就是类方法（只有这个类能够调用的方法，不就是这个类的单件方法吗？）
以下简述定义类的单件方法（类方法）的三种方式：

1. 不推荐，因为显式重复了类名，会给重构带来不便

    ```ruby
    def Person.class_method
        # do something
    end
    ```

2. 比较常用，在类体中用 self 指明接受者作为类本身，本质上和第一种方法相同

    ```ruby
    class Person
        self.class_method
            # do something
        end
    end
    ```

3. 比较常用，但是意义与上两种完全不同，虽然和第二种一样定义在类体中，但是 `class <<` 方法直接打开了该类的单件类的作用域。这种方式在定义大量类方法的时候非常方便。

    ```ruby
    class Person
        class << self
            def class_method
                # do something
            end
        end
    end
    ```
其实还有第四种方式可以定义类方法，不过那种方法并不符合设计本意，如果你想了解，请看 *"instance_eval 和 class_eval"* 这篇文章。

---

> 单件类 和 单件方法 的内容远不止这些，对象的单件类 和 类的单件类 在性质上也有很大的不同。我这里只是做一个非常基础的介绍，帮助你最快速的认识什么是单件类和单件方法，并且让你能够开始使用它们。