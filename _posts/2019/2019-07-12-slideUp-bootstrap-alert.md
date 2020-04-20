---
layout: post
title: 用 slideUp 方式移除 bootstrap 的 alert
date: 2019-07-12
tags: javascript
---

Bootstrap 的 alert 只提供了 fade 方式，如果我们想改变 alert 的移除样式，比如用 slideUp 方式，该如何设置呢？

首先来看看 Bootstrap alert 的 [Dismissing](https://getbootstrap.com/docs/4.1/components/alerts/#dismissing)，官网的例子是：

```html
<div class="alert alert-warning alert-dismissible fade show" role="alert">
  <strong>Holy guacamole!</strong> You should check in on some of those fields below.
  <button type="button" class="close" data-dismiss="alert" aria-label="Close">
    <span aria-hidden="true">&times;</span>
  </button>
</div>
```

Bootstrap 默认提供了 fade 来增加 alert 的移除效果，如果要改成别的移除样式，就很麻烦，这是为什么呢？

理论上，只要不在 class 里加上 fade，然后自己在 js 里写 `$(".alert").slideUp("slow");` 就可以了。但结果是，alert 在 slideUp 的效果结束后瞬间又出现了，即使添加了一个 `style="display: none"`。麻烦就在于，bootstrap 提供了专门的 [triggers](https://getbootstrap.com/docs/4.1/components/alerts/#triggers) 来手动关闭 alert，用普通的元素删除指令是无法删除 alert 的，slideUp 只提供了移除的样式，而 .alert('close') 才是真正将 alert 从页面移除的指令。

所以，我们只需要在 slideUp 后调用 .alert('close') 来彻底删除 alert。但仅仅把这两条命令写在一起就可以了吗？问题没那么简单。

因为 slideUp 是一个渐变的效果，是有延迟的，虽然两句话一前一后看起来有顺序，但是 javascript 是单线程的，命令是同步运行的，后一句并不会等到上一句执行结束才会开始，程序的同步运行是会不停往下跑的。所以，如果单单把两句话一前一后写在一起，结果就是 alert 会直接消失，没有 slideUp 的效果。

因此，最简单的方法，就是设置一个 setTimeout 让 .alert('close') 等到 slideUp 结束再执行，slideUp("slow") 执行时间是 600ms，所以只要设置 600ms 的延迟就行了。

还有一个更高级也更麻烦的用法，就是通过 Promise 对象来实现异步，参考阮一峰的[《ECMAScript 6》](http://es6.ruanyifeng.com/#docs/promise)。

最后，再用 setTimeout 来模拟 3 秒后自动通过点击 x 来关闭 alert 框来实现自动消失。

### 代码如下

```html
<% if alert %>
  <div class="alert alert-danger alert-dismissible d-flex justify-content-center" role="alert">
    <button type="button" class="close js-alert-close" daria-label="Close">
      <span aria-hidden="true">&times;</span>
    </button>

    <h5><%= alert %></h5>
  </div>
<% end %>

```
* 注：别忘了删除 button 的 `data-dismiss="alert"`

```javascript
if($(".alert").length > 0){
  setTimeout(function(){    
    $(".alert").find(".js-alert-close").click();
  }, 3000);
}

$(".js-alert-close").click(function(){
  $alert = $(this).closest(".alert");
  $alert.slideUp("slow");
  setTimeout(function(){
    $alert.alert('close');
  }, 600);
  // let promise = new Promise(function(resolve, reject){
  //   $alert.slideUp("slow");
  //   setTimeout(function(){
  //     resolve();
  //   }, 600);
  // })
  // promise.then(function(){
  //   $alert.alert('close');
  // })
});
```

---

## 参考资料：

* [Bootstrap Alert](https://getbootstrap.com/docs/4.1/components/alerts/)
* [《ECMAScript 6》- Promise](http://es6.ruanyifeng.com/#docs/promise)