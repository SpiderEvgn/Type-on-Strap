---
layout: post
title: React + Express 搭起简易微信登录框架（五）
tags: [react, express, wechat]
---

## 0. 引言

上一篇讲到，我这个 Demo 的登录跳转是有个『bug』的，它到底是什么呢？

***[Demo 的完整代码在 [这里](https://github.com/SpiderEvgn/react-wechat-login-demo)]***

## 1. 放大问题

先来加一点代码把问题放大，便于观察。

来到 app_server.js，添加如下函数：

```js
// 手动模拟一个延时
const sleep = ms => (
  new Promise(res => setTimeout(res, ms))
)
```

然后在响应路由中加一句：

```js
await sleep(5000)
```

这样就模拟了一个后台 5秒延时的响应，比如网络卡顿了，在请求微信 API 的时候卡了 5秒。

然后清空 cookie，再次点击『微信一键登录』，观察浏览器的跳转流程：

1. 来到 Login 页面并点击『微信一键登录』：「http://192.168.0.101:3000/login」
2. Login 中拼接的微信地址：「https://open.weixin.qq.com/connect/oauth2/authorize?appid=...」
3. 带 code 的回调地址：「http://192.168.0.101:3000/?code=...」
4. 回到登录页：「http://192.168.0.101:3000/login」
5. 等待 5秒后进到根页面：「http://192.168.0.101:3000/」

停了 5秒后我们就能很明显地体会到这个很诡异的等待了，这是因为微信回调访问根页面的时候还没登录，于是登录页就直接显示出来了，渲染完成后执行 AuthorizedRoute 的 useEffect，发起异步请求到后端完成 openid 的请求，再把结果传回前端执行 dispatch，最后更新状态再次刷新登录页，发现是已经登录的状态，于是跳转到根页面。

## 2. 怎么办？

这个问题本身其实是很容易解决的，只需添加一个等待状态，在这 5秒内告诉用户后台正在验证登录，比方说登录按钮显示『正在登录。。。』然后变为不可点击，等等。

但其实我想说的并不是这个解决方案。不过，先让我们把这步实现，让整个登录流程变得通顺一点，然后再来看看还有没有什么别的问题。

首先，给 axios 添加一个拦截器，在 useEffect 中异步给 Express 发出请求的时候给页面添加一个等待状态。

创建 `src/utils/Axios.js` 文件：

```js
import React from 'react'
import ReactDOM from 'react-dom'
import axios from 'axios'

const Axios = axios.create({
  timeout: 10000
})

const showLoading = () => {
  var dom = document.createElement('div')
  dom.setAttribute('id', 'loading')
  document.body.appendChild(dom)
  ReactDOM.render(<p>正在登录。。。</p>, dom)
}

const hideLoading = () => {
  document.body.removeChild(document.querySelector('#loading'))
}

Axios.interceptors.request.use(config => {
  showLoading()
  return config
})

Axios.interceptors.response.use(res => {
  hideLoading()
  return res
}, err => {
  hideLoading()
  return Promise.reject(err)
})

export default Axios
```

这里我只简单的在页面中央打印了『正在登录。。。』，效果如下：

![loading](/assets/img/posts/2020/react-wechat/loading.jpg "loading")

接下来，到 AuthorizedRoute 组件更改 axios 的引用库：

```js
import axios from 'axios'
// 更改为 --->
import axios from 'utils/Axios'
```

到此异步等待的页面就实现了，交互顺畅了很多，一切看起来都很美好。

## 3. 异步带来的真正问题

但其实我真正想讨论的，是这个等待功能实现之后，依旧存在的一个跳转问题。

这个 Demo 并没有实现这样的逻辑，但我们想象一下，如果登录后还有别的中间逻辑，比如绑定微信 openid 与本地用户系统，当用户首次登录的时候需要展现给他这样一个绑定操作界面，而已绑定的用户再次登录时直接进入内容界面。

这个功能只是举个例子，请忽略具体的需求。

抽象来说，用户当前在 A 页面，点击某个按钮后前往 B 页面，但是 B 页面有个前置逻辑判断，满足一定条件则显示 C 页面。而根据 useEffect + axios 的异步请求特性，不论条件满足于否，B 页面一定会渲染，如果满足则再跳转到 C，所以结果就是在 A -> C 的过程中 B 一定会闪现。

我想说的是，可不可以同步 useEffect，即在渲染页面前先进行逻辑判断，满足一定条件则直接跳转页面，而不是先把组件渲染到页面，等异步判断完成更新状态后再跳转，这样就会出现页面闪现的情况。

实际上，React 是考虑到这个逻辑先于渲染的需求的，通过另一个 hook 就可以实现：useLayoutEffect。参考[这篇文章](https://daveceddia.com/useeffect-vs-uselayouteffect/)，可以简单理解为 useLayoutEffect 就是同步版本的 useEffect，React 会等待 useLayoutEffect 执行完毕再渲染。

这看上去非常完美，于是马上激动地换上 useLayoutEffect，发现结果并没有变化。React hook 出 bug 了？

当然不是。useLayoutEffect 的确是在渲染前同步执行了，问题出在 axios 请求。尽管 useLayoutEffect 先于渲染执行了，但是 axios 仍然是异步的，它仍然进入了异步队列，页面渲染完成后再慢悠悠地执行起来。

给 axios 加 async/await！就像我们在 Express 中做的那样。

首先，useEffect 方法不能直接用 async，像下面这样是不合法的，React 会报错：

```js
useEffect(async () => {
  // some logic
  await ...
})
```

useEffect 里使用 async 的话只能再套一层，比如用 IIFE(Immediately Invoked Function Expression) 的方式：

```js
useEffect(() => {
  (async () => {
    // some logic
    await axios...
  })()
})
```

但结果令人失望，这并不能同步 axios 请求。

多番尝试，这似乎是一个死胡同。

## 4. 官方出手了

终于，我最后搜到了官方给出的一个解决方案（原来 React 团队已经着手解决这个问题了），不过还不算正式发布，只是在试用阶段。

它就是 [Suspense](https://zh-hans.reactjs.org/docs/concurrent-mode-suspense.html)。

Suspense 的用法不在这里讨论，它还涉及到很多问题，我也还在研究中，不过，当我得知官方给出 Suspense 的时候，我的好奇心已经得到了满足。

## 5. 总结

其实扯到 Suspense 已经和我的 Demo 没多大关系了，我的 Demo 并未涉及到这个问题。

在此延伸出来讲，是想稍微表达一些个人观点。

前端的世界变化快速，各种理念和技术都日新月异，在以 React 为代表的这些新时代 UI 技术颠覆传统前端的同时，也不免伴随许多新生的问题。原来在 JQuery 时代的很多痛点的确在被一一解决，但随之而来的也有本来可以轻易实现（或许不那么美）的功能在新的理念下要完全脱胎换骨。

得与失不是轻易说的清的。而从需求出发永远是真理，适合自己的才是最好的。

但话说回来，我相信新时代的前端发展方向是未来，相比于古老的 js 代码，如今的前端框架越来越结构化，越来越松耦合，语法也越来越漂亮，舞台也一定会越来越大。