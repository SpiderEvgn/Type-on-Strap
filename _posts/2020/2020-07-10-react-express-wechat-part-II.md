---
layout: post
title: React + Express 搭起简易微信登录框架（二）
tags: [react, express]
---

## 0. 引言

这篇来讲如何搭建一个 Express 后台，前端开发环境如何配置代理，以及生产环境的应用。

上一篇在初始化项目的时候，已经安装好了 [Express](http://expressjs.com/) 和 [http-proxy-middleware](https://create-react-app.dev/docs/proxying-api-requests-in-development)，现在就来启动一个 Express 后台。

***[Demo 的完整代码在 [这里](https://github.com/SpiderEvgn/react-wechat-login-demo)]***

## 1. Express 服务器

我的 Demo 对后台的要求非常简单，只是作为一个 API 的请求中转，向微信拿 openid，仅此而已。至于其他的业务操作不在本文讨论范围。

所以，本文建立的是一个最简化的 Express 后台，只包括最简单的请求处理能力，一个文件，几行代码，就搞定了。

在项目根目录下创建 app_server.js:

```js
const express = require('express')
const app = express()

app.get('/api/fetch-wechat-userinfo', (req, res) => {
  return res.json({
    status: 'ok',
  })
})

app.listen(3005, () => {
  console.log('App is listening on port 3005...')
})
```

服务简单到 "令人发指"：导入 express，写一个 route，然后在 3005 端口启动。

## 2. 前端代理

因为前端请求有跨域问题，不能直接发起 ajax 请求，所以采用[官方推荐的做法](https://create-react-app.dev/docs/proxying-api-requests-in-development/)，在开发环境配置代理（生产环境稍候介绍）。

`http-proxy-middleware` 已经安装完成，直接在 src 目录下创建 setupProxy.js 文件（文件名是约定好的）：

```js
const { createProxyMiddleware } = require('http-proxy-middleware')

module.exports = function(app) {
  app.use(
    '/api', 
    createProxyMiddleware({
      target: 'http://localhost:3005',
      changeOrigin: true
    })
  )
}
```

这个配置表示，代理服务会把所有 `/api` 开头的请求代理到本机 3005 端口即我们的 Express 服务。

然后我们就可以从前端发个测试请求，来验证一下整个流程了。

## 3. 验证请求

在 AuthorizedRoute 组件中加入一个 `useEffect hook` 如下：

```js
// 别忘了引入 useEffect 和 axios 库
import React, { useContext, useEffect } from 'react'
import axios from 'axios'

...

const AuthorizedRoute = ({ children, ...rest }) => {

  ...

  useEffect(() => {
    axios(`/api/fetch-wechat-userinfo`)
      .then(res => console.log(res.data))
  })

  ...
```

然后启动 Express 服务：

```bash
node app_server.js
```

当我们再次访问 '/' 页面的时候，发现控制台输出 `{status: "ok"}`，后台响应测试成功。

接下来我们要做的，就是在这个后台响应中，完成真正的微信 API 请求。

## 4. 生产环境怎么办？

React 官方只提供了开发环境的代理模式，却只字未提生产环境的配置，网上也几乎没有什么有用的信息，生产环境也的确无法使用这个代理的方法。

我的生产环境是 nginx 直接使用打包后的静态文件，当然与开发环境的 `npm start` 不同，但我转念一想，后端的 Express 也只是一个服务而已，功能也很简单，只是转发微信 API，那只要在 nginx 匹配一下路由把 `/api` 请求转发到 3005 服务不就完了。

于是改写后的 nginx 配置如下：

```bash
location ^~ /api/ {
  proxy_redirect off;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-Forwarded-Host $host;

  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";

  proxy_pass http://localhost:3005;
}

location / {
  try_files $uri $uri/ /index.html;
}
```

这样，所有以 `/api` 开头的请求都会转发到 3005 端口的服务（后端 Express 服务），其余仍然用打包后的静态文件，成功地拆分了不同的服务。

## 5. 总结

Demo 只涉及微信手机客户端一键登录，功能非常简单，所以用到的 Express 也非常简单，而且没有设置数据库，后端分离的话可以扩展一下 api 把请求转发给真正的后端应用服务器（不是 Demo 的 Express），而复杂的应用就要彻底改一下架构了。

下一篇正式进入微信登录功能，注意是手机端通过微信客户端登录第三方网站的方式（网页授权）。