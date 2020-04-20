---
layout: post
title: Flex 布局
date: 2019-07-13
tags: css
---

这篇文章讲一讲利用 Flex 的布局（基于 [Bootstrap 的 Flex 包装]((https://getbootstrap.com/docs/4.1/utilities/flex/)) 包装）。关于 Flex 的具体用法，可以参考[阮神的文章](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)。

### 需求案例

![whole-element](/assets/img/posts/2019/vertical-center/whole-element.png "Whole Element")

需要完成的内容是：一个 block 由两部分组成，左边的 icon 和标题文字，置右的按钮。因为按钮 height 更大撑高了空间，所以左边的三角 icon 和标题字段需要垂直居中。此外，点击整个 block 的时候需要切换三角 icon 为打开/关闭，即箭头朝右和朝下两个 icon，问题是两个 icon 的宽度不同，箭头朝右偏大一点，切换时会影响到标题文字，所以需要将标题文字固定在一定位置不受 icon 宽度变化的影响。

### 具体实现思路

* 首先，因为需要整体居中，所以将整个 h5 设置成 flex，然后用 align-items-center 垂直居中。（实际上这步只是将 icon 居中了，因为剩余部分已经撑满了高度）

  ![h5](/assets/img/posts/2019/vertical-center/h5.png "H5 Element")

* 因为不能让三角 icon 的宽度变化影响到标题字段，所以 icon 不给宽度，而给标题文字一个固定的左边距。

  *注：这里不能用 float，因为在 flex 布局内 float 是无效的*

  ![fas-caret](/assets/img/posts/2019/vertical-center/fas-caret.png "Fas Caret")

* icon 剩下的部分作为整体，给一个左边距 `ml-4`，然后设置成 flex，同样用 align-items-center 将标题文字垂直居中（同样因为按钮比较大撑满了高度），最后再用 flex-grow-1 把宽度撑满（为什么要撑满？最后再解释）。

  ![content-after-caret](/assets/img/posts/2019/vertical-center/content-after-caret.png "Content After Caret")

* 将在标题文字后的部分设置成 flex，然后用 flex-grow-1 将宽度撑满，最后再用 flex-row-reverse 将按钮置右

  ![float-right-button](/assets/img/posts/2019/vertical-center/float-right-button.png "Float Right Button")

  > 现在回答将 icon 剩余部分撑满宽度的理由。因为要把按钮置右，所以标题后的部分宽度必须是满的，否则按钮置右也只是紧挨在标题文字后面。同时，为了将标题后的部分撑满宽度，倒推上去就必须将它的父元素撑满宽度，也就是 icon 剩余的整个部分。

### 上完整代码

```html
<div class="card-header" id="care_header">
  <h5 class="d-flex align-items-center">
    <i class="fas fa-caret-down js-caret width-0"></i>
    <div class="ml-4 d-flex align-items-center flex-grow-1">
      测试标题字段
      <div class="d-flex flex-grow-1 flex-row-reverse">
        <button class="btn btn-outline-primary ml-2" type="button">
          <i class="fas fa-plus"></i>
           测试按钮2号
        </button>
        <button class="btn btn-outline-success" type="button">
          <i class="fas fa-history"></i>
           测试按钮1号
        </button>
      </div>
    </div>
  </h5>
</div>
```

## 参考资料：

* [Bootstrap 的 Flex 包装](https://getbootstrap.com/docs/4.1/utilities/flex/)
* [阮神介绍 Flex](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)



