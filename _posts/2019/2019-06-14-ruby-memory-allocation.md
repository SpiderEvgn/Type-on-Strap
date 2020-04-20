---
layout: post
title: 【翻译】是什么造成了 Ruby 的内存膨胀？
date: 2019-06-14
tags: ruby
---

> 这是一篇翻译文，作者对 Ruby 的内存回收进行了深入的研究，并且讲到了操作系统的内存回收，写的很好，所以我将它翻译到此，这是[原文地址](https://www.joyfulbikeshedding.com/blog/2019-03-14-what-causes-ruby-memory-bloat.html)，以下是正文。

在 [Phusion](https://www.phusionpassenger.com/) 我们运行着一台用 Ruby 写的简单的多线程 HTTP 代理服务器（用来伺服 DEB 和 RPM 包），我发现它占用了 1.3 GB 的内存。实在是太夸张了，这台服务器是无状态的并且根本没有那么多的工作消耗！

![ruby-mem-graph-doodle](/assets/img/posts/2019/ruby-memory/ruby-mem-graph-doodle.jpg "Ruby Mem Graph Doodle")
        *问：这是什么？ 答：一个 Ruby 进程的持续内存使用情况！*

实际上并非只有我一人遇到了这个问题，Ruby 的应用就是会占用大量的内存。但，这是为什么呢？根据 [Heroku](https://devcenter.heroku.com/articles/tuning-glibc-memory-behavior) 和 [Nate Berkopec](https://www.speedshop.co/2017/12/04/malloc-doubles-ruby-memory.html) 的说法，大部分过量的内存占用是由于 **内存碎片** 和 **内存过度分配** 导致的，

Nate Berkopec 总结了两个解决方案：

  1. 用一个与 glibc 完全不同的内存调度器，通常是 [jemalloc](http://jemalloc.net/)
  2. 设置神奇的环境变量： `MALLOC_ARENA_MAX=2`

问题的描述和这两个解决方案都令我感到困惑。我并不觉得这个问题的描述是完全正确的，或者这是唯一的解决方法，同时我也很吃惊有你那么多人[把 jemalloc 当作是一个神奇的银弹](https://medium.com/rubyinside/how-we-halved-our-memory-consumption-in-rails-with-jemalloc-86afa4e54aa3)

**魔法仅仅是我们还未理解的科学。** 所以我开始了这次研究，试图找到背后真正的原因，这篇文章将会囊括：
  
  1. 内存分配的介绍：它是如何工作的？
  2. 什么是人们常说的『内存碎片』和『内存过度分配』？
  3. 什么造成了内存的高度占用？这个问题是人们提到的仅此而已吗？或者还有更深的影响？（提示：是的，我会分享我的研究结果）
  4. 有没有其他的替代解决方案？（提示：我找到了一个）

*注：这篇文章的内容仅适用于 Linux 系统及多线程的 Ruby 应用*

### 目录：

* Ruby 内存调度 101
  * Ruby
  * 系统内存调度器
  * 内核
  * 内存使用定义
* 什么是碎片？
  * Ruby 层的碎片
  * 内存调度器层的碎片
* Ruby 堆页（heap pages）层的碎片是内存膨胀的原因吗？
* 调查内存调度器层的碎片
  * 过度分配和 glibc 内存竞技场
  * 观察操作系统堆
* 一个神奇的小技巧：修剪（trimming）
* 总结
  * 观察器源码
  * 性能如何？


### Ruby 内存调度 101

Ruby 的内存调度有三层，从高到低排序：
  1. Ruby 解释器，用来管理 Ruby 对象
  2. 操作系统的内存调度器库
  3. 内核

我们依次来看。

### Ruby

在 Ruby 端，Ruby 在内存中管理对象的区域叫做 **Ruby 堆页（Ruby heap pages）**。Ruby 堆页被分成大小相等的槽位，一个对象占据一个槽位。不管这个对象是字符串、哈希表、数组、类，或任何什么，都占据一个槽位。

![ruby_heap_page](/assets/img/posts/2019/ruby-memory/ruby_heap_page.png "Ruby Heap Page")

Ruby 堆页的槽位的状态只有「被占据」和「空闲」两种。当 Ruby 创建了一个新的对象，它会先尝试指定一个空闲的槽位，当没有空余时，会分配一个新的堆页。

一个槽位是很小的，大约 40 bytes，显然不是所有 Ruby 对象都能装进去，比如一个 1 MB 的字符串。Ruby 处理这个问题的方式是把超过 40 bytes 的对象存储在堆页外部一个不同的地址空间，然后 Ruby 通过在槽位放置一个指针来找到这个存储在外部的数据。

![ruby_extra_data](/assets/img/posts/2019/ruby-memory/ruby_extra_data.png "Ruby Extra Data")

Ruby 堆页和任何外部的数据都是用系统的内存调度器分配管理的。

### 系统内存调度器

操作系统的内存调度器是 glibc（C 的运行时）的一部分，不仅仅是 Ruby，它几乎被用在所有应用上。它有一个简单的 API：

  * 通过调用 `malloc(size)` 分配内存，传入需要分配的 bytes，返回结果地址或者是一个报错。
  * 通过调用 `free(address)` 回收内存。

不同于 Ruby 在大部分情况下使用相同大小的槽位，内存调度器必须能够处理各种大小的内存分配请求。之后会讲到，这个特点会带来一定的复杂度。

内存调度器通过内核 API 依次分配内存空间，它从内核获取比请求地大得多的内存块，因为调用内核开销很大，而且内核 API 有一个限制：它只能以 4 KB 为单位分配内存。

![memory_allocator_alloc](/assets/img/posts/2019/ruby-memory/memory_allocator_alloc.png "Memory Allocator Alloc")

内存调度器从内核获取的内存空间叫做 **堆（heap）**，注意这个「堆」和「Ruby 堆页」没有任何关系，所以为了辩识之后我会称之为「操作系统堆」。

接着，内存调度器会一块一块地将操作系统堆分配给它的调用者，直到空间用完，然后它再从内核申请新的操作系统堆。这和 Ruby 从 Ruby 堆页分配槽位给对象的过程很像。

![memory_alloc_interactions](/assets/img/posts/2019/ruby-memory/memory_alloc_interactions.png "Memory Alloc Interactions")

### 内核

内核只能以 4 KB 为单位分配内存，这样一个 4 KB 的空间叫做 **页（page）**。为了不和「Ruby 堆页」搞混，之后我会称之为「操作系统页」。

之所以有这个特性的原因很复杂，但所有现代的内核都是如此。

通过内核分配内存会有巨大的性能开销，所以内存调度器要尽可能减少内核调用的次数。

### 内存使用定义

我们知道了内存是在多个层面上进行调度分配的，并且每个维度分配的内存空间都是大于实际需求的。Ruby 堆页可以拥有空闲的槽位，操作系统堆也可以有空闲的块，所以当你问『使用了多少内存？』的时候，答案其实取决于你在哪个层面问这个问题！

比如说，当你用 `top/ps` 命令工具去检查一个进程的内存使用情况，返回的是内核层的使用量。这意味着从内核的角度来看，高于内核的层面必须一起合作来释放内存。正如你将在下一节看到的关于碎片的内容，这个过程也许比看起来要复杂得多。

### 什么是碎片？

分配的内存散落在各个地方，这种现象被称作内存碎片，它会造成许多有趣的问题。

### Ruby 层的碎片

考虑一下 Ruby 的垃圾回收机制。Ruby 对象的垃圾回收指的是把一个 Ruby 堆页槽位标记成空闲，即允许该槽位被再次使用。如果一整个 Ruby 堆页的槽位全部都是空闲的，则这个 Ruby 堆页就可以被内存调度器收回了（然后收回到内核）。

![ruby_heap_fragmentation](/assets/img/posts/2019/ruby-memory/ruby_heap_fragmentation.png "Ruby Heap Fragmentation")

但，如果不是所有槽位都是空闲的呢？如果有许多 Ruby 堆页，然后一个垃圾回收任务在不同的位置释放了对象空间，结果是你得到了许许多多的空闲槽位，而每个 Ruby 堆页的槽位却不全是空闲的？这种情况下，即便 Ruby 仍然拥有许多空闲的槽位可以分配给新对象，但在内存调度器和内核看来，这些都是已经占用的内存空间！

### 内存调度器层的碎片

内存调度器有类似却又不同的问题。内存调度器并不需要一次性释放整个操作系统堆，理论上，它可以释放任何单个操作系统页，但是因为它不得不处理所有大小的内存分配，所以一个操作系统页可能包含多个不同的内存分配，结果是只有当一个操作系统页中的所有内存分配都被释放了之后内存调度器才可以回收它。

![os_page_partial_free](/assets/img/posts/2019/ruby-memory/os_page_partial_free.png "OS Page Partial Free")

比方说，如果有一个 3 KB 和一个 2 KB 的内存空间都分布在两个操作系统页上，即使你释放了这个 3 KB 的内存空间，第一个操作系统页仍然是有一部分被占用着的，所以它还是不能被回收。

![os_page_fragmentation](/assets/img/posts/2019/ruby-memory/os_page_fragmentation.png "OS Page Fragmentation")

所以如果我们不够幸运，则可能即便在操作系统堆里面拥有许许多多的空闲空间，但仍然缺乏满足需求的完全空闲的操作系统页。

更糟糕的情况是，如果有很多的内存洞，但没有一个空间是满足新的内存请求的，那么内存调度器将不得不申请一个新的操作系统堆。

### Ruby 堆页（heap pages）层的碎片是内存膨胀的原因吗？

碎片是导致 Ruby 高度内存消耗的原因，这句话看起来是很有道理的。我们先假设碎片的确就是这个原因，那这两个碎片的源头哪一个才是最主要的元凶呢？

  1. Ruby 堆页碎片
  2. 内存调度器碎片

有一个很简单的方法来判断是否是第一个。Ruby 提供了两个 API：`ObjectSpace.memsize_of_all` 和 `GC.stat`。通过两者的返回结果，我们能够计算 Ruby 已知地从内存调度器申请的所有内存空间。

![ruby_memsize_of_all](/assets/img/posts/2019/ruby-memory/ruby_memsize_of_all.png "Ruby Memsize Of All")

`ObjectSpace.memsize_of_all` 返回所有活动的 Ruby 对象占用的内存，也就是所有槽位和外部数据的占用空间。如上图所示，就是所有蓝色和橙色对象的空间。

`GC.stat` 可以计算所有空闲的槽位，比如上图所有灰色的区域。这是它的算法：

```ruby
GC.stat[:heap_free_slots] * GC::INTERNAL_CONSTANTS[:RVALUE_SIZE]
```

如果把它们汇总，则这就是 Ruby 已知地占用的所有内存大小，包括了 Ruby 堆页的碎片。这就是说，如果这个进程的内存使用大于该值，那么多出来的这部分内存大小来自于 Ruby 所不能控制的某个地方，比如第三方库或者别的碎片。

我写了一个简单的测试程序用来制造一堆线程，每一个线程循环创建字符串。下图是一段时间后我测量得到的内存使用量：

![ruby_mem_usage_test](/assets/img/posts/2019/ruby-memory/ruby_mem_usage_test.png "Ruby Mem Usage Test")

这...简直太草泥马了！

结果显示 Ruby 自己的内存消耗只占所有内存使用量的很小一部分，Ruby 的堆页是否碎片根本无关痛痒。

我们必须去别的地方找这个罪魁祸首，至少现在我们知道了这不是 Ruby 的错！（译者加：谢天谢地！）

### 调查内存调度器层的碎片

另一个嫌疑者是内存调度器。毕竟，Nate Berkopec 和 Heroku 都说过摆弄内存调度器（整个替换成 jemalloc，亦或是设置神奇的环境变量 `MALLOC_ARENA_MAX=2`）可以大幅度降低内存消耗。

我们首先来看看什么是 `MALLOC_ARENA_MAX=2` 并且它是如何起作用的。接着，我们再研究内存调度器层是否碎片，并且有多大程度的碎片。

### 过度调度和 glibc 内存竞技场

`MALLOC_ARENA_MAX=2` 起作用的原因和多线程密不可分。当多个线程尝试同时从同个操作系统堆获取内存时，它们会互相竞争。因为一次只有一个线程可以得到内存分配，所以降低了多线程的内存分配的性能。

![os_heap_contention](/assets/img/posts/2019/ruby-memory/os_heap_contention.png "OS Heap Contention")

对于这种情况内存调度器有一个优化方案，它尝试创建多个操作系统堆然后给不同的线程指定一个它自己的操作系统堆。大部分时候一个线程只需要一个操作系统堆，因此避免了多线程的竞争。

事实上，操作系统堆以这种方式分配的最大数量默认是虚拟 CPU 数量的 8 倍。所以在一个 2 核每核 2 个超线程的系统上，一共可以有 2 * 2 * 8 = 32 个操作系统堆！这就是我说的 **过度分配**。

为何默认的数量如此巨大？是因为内存调度器的主要开发是 Red Hat，它的客户都是那些拥有巨量 RAM 机器的企业。上面这种优化方法能够在巨大的内存使用情况下提高平均 10% 的多线程性能。对于 Red Hat 的客户来说，这是利大于弊的，而对于大部分非企业用户来说则不然。

Nate 的博客和 Heroku 的文章都申明操作系统堆等于更多的碎片，并引用了官方文档。`MALLOC_ARENA_MAX` 变量减少了分配给为了降低多线程竞争的操作系统堆的最大数量，以此降低了碎片。

### 观察操作系统堆

Nate 和 Heroku 关于更多的操作系统堆等于更多碎片的断言是正确的吗？实际上碎片完全是内存调度器层的问题吗？我不想妄言任何一个猜测，所以我开始了自己的研究。

如果有一个方法可以观测操作系统堆就好了，那样我就可以观察到底发生了什么。不幸的是，**没有工具可以做到这点**。

# 所以我自己写了一个
# 操作系统堆的观测器

首先，我必须想办法将操作系统堆的排列布局收集下来。所以我深挖了[内存管理器的源代码](https://github.com/bminor/glibc/tree/master/malloc)试图找到内存管理器内部是如何表示内存分配的。接着，我写了一个库用来理清这些数据结构并将结果写入一个文件。最后，我还写了一个工具，将这个文件作为输入，输出一个 HTML 和 PNG 图片组成的可视化展现。[这是源代码](https://github.com/FooBarWidget/heap_dumper_visualizer)。

![os_heap_visualization](/assets/img/posts/2019/ruby-memory/os_heap_visualization.png "OS Heap Visualization")

这是一个操作系统堆的可视化例子（还有很多），小格子代表了操作系统页。

  * 红色的区域是占用的内存空间
  * 灰色的区域是空闲的空间，但尚未回收到内核
  * 白色的区域已经回收到内核

从该页面我可以得出如下结论：

  1. 这里有碎片。因为红色的部分是分散的，而且有些操作系统页只有一部分是红色的
  2. 令我感到惊奇的是，大部分操作系统堆看起来是：巨量的毫无红色的完整的灰色操作系统页！

然后，我就懵逼了：

# 感觉好像碎片是个问题，但它看上去其实并没有想象的那么糟糕！

反而那些巨量的灰色看起来更有问题：这说明内存调度器并没有把内存释放回内核！

在更深地研究了内存调度器的源代码后，我发现默认情况下它只释放操作系统堆最末端的操作系统页，而且这是 **偶然的**。这也许是出于性能的考虑。

### 一个神奇的小技巧：修剪（trimming）

很幸运我发现了一个神奇的小技巧。有一个 API 可以强制内存调度器将所有符合条件的操作系统页回收到内核，不仅仅是那些末端的。它叫做[malloc_trim](http://man7.org/linux/man-pages/man3/malloc_trim.3.html)。

我以前知道这个函数但并没想到它如此有用，因为它的手册有如下一句话：

> `malloc_trim()` 函数试图释放堆顶部的内存空间。

**MMP 手册是错的！** 通过对源码的分析我了解到它实际上释放了所有合适的操作系统页，不仅仅是那些头部的。

所以我在想，如果我们在 Ruby 垃圾回收的时候调用这个函数会发生什么？我修改了 Ruby2.6 在 gc.c 调用 `malloc_trim()`，函数 `gc_start`，如下所示：

```
gc_prof_timer_start(objspace);
{
    gc_marks(objspace, do_full_mark);
    // BEGIN MODIFICATION
    if (do_full_mark)
    {
        malloc_trim(0);
    }
    // END MODIFICATION
}
gc_prof_timer_stop(objspace);
```

下图是测试结果：

![after_malloc_trim](/assets/img/posts/2019/ruby-memory/after_malloc_trim.png "After Malloc Trim")

**结果完全不同了！**这个简单的补丁导致了内存使用量几乎核设置 `MALLOC_ARENA_MAX=2` 一样低。

下图是可视化界面：

![os_heap_visualization_after_trim](/assets/img/posts/2019/ruby-memory/os_heap_visualization_after_trim.png "OS Heap Visualization After Trim")

我们可以看到许多「白洞」，它们是已经回收到内核的操作系统页。

### 总结

碎片其实更像是一条红鲱鱼，通过降低碎片的确能有所收获，但主要的问题出在内存调度器并不会将内存释放回内核。

这个结论非常简单，但是过程非常痛苦...

### 观察器源码

[这是观察器源代码](https://github.com/FooBarWidget/heap_dumper_visualizer)。

### 性能如何？

一个很大的问题是性能。调用 `malloc_trim()` 不可能是零损耗的，通过查看代码这个算法是线性复杂度的。所以我找到了[Noah Gibbs](https://twitter.com/codefolio)（译者注：twitter 链接，国内...你懂的），运行 Rails Ruby Bench 的那个人。最后神奇的是他发现我的补丁居然还些微地提升了性能。

![noah_gibbs_performance_quote](/assets/img/posts/2019/ruby-memory/noah_gibbs_performance_quote.jpg "Noah Gibbs Performance Quote")

![wait_what](/assets/img/posts/2019/ruby-memory/wait_what.jpg "Wait What")

我又懵逼了。我完全无法解释，但这的确是个好消息。
