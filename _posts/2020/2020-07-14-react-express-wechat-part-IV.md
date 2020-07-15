---
layout: post
title: React + Express 搭起简易微信登录框架（四）
tags: [react, express, wechat]
---

## 0. 引言

本文将引入 cookie 机制，保存用户的登录状态，有效期内访问不用重复发起微信 API 请求再次获取 openid。

因为我的 Demo 其实是没有后端的（Express 仅仅是为了微信请求），所有用户系统、openid 的绑定和识别等等都不在 Demo 的范围（按需另行添加）。

***[Demo 的完整代码在 [这里](https://github.com/SpiderEvgn/react-wechat-login-demo)]***

## 1. 安装 cookie-parser

[cookie-parser](https://github.com/expressjs/cookie-parser#readme) 是 Express 官方的 cookie 包，直接使用它来处理 cookie 操作。

```bash
yarn add cookie-parser
```

## 2. Express 设置 cookie

在第一次获取到 openid 后，就需要通知前端浏览器记住当前用户，当下一次用户再访问的时候就能通过 cookie 判断了。

为浏览器设置 cookie 的时候不推荐直接使用 js 在客户端操作，而应该在后端响应中带上 set-cookie 头，即在 Demo 的 Express 服务中设置。

为 app_server.js 增加如下代码：

```js
// 导入库
const cookieParser = require('cookie-parser')

// 在路由前添加
app.use(cookieParser('secret-openid'))
app.get('/api/fetch-wechat-userinfo', async (req, res) => {

  ...

  // 最后在返回结果前存储 cookie，将 openid 加密存储到 token 关键字
  if(openid !== 'error'){
    res.cookie('token', openid, {
      maxAge: 7 * 24 * 60 * 60 * 1000,    // 可以设置得短一点测试有效期，比如 30 秒（30 * 1000）
      signed: true
    })
    return res.json({
      status: 'ok',
      openid: openid
    })
  }else{
    return res.json({
      status: 'error'
    })
  }
```

验证一下，重启 app_server.js，再次点击『微信一键登录』，在浏览器的 Application -> Cookies 中出现了名为 token 的 cookie，成功。

## 3. 前端判断 cookie

cookie 设置完成后，就需要在每次访问时都读取并判断。

首先写一个获取到 cookie 的方法，创建 `src/utils/utils.js` 文件：

```js
export const getCookie = (key) => {
  var name = key + '='
  var ca = document.cookie.split(';')
  for(let i=0; i < ca.length; i++){
    let c = ca[i].trim()
    if(c.indexOf(name)===0) return c.substring(name.length, c.length)
  }
  return ''
}
```

然后，在什么地方判断这个登录 cookie 呢？AuthorizedRoute 保护路由显然不行，因为登录页面属于公开路由，登录后再次进入登录页面就很不合理。所以应该有个地方把所有路由都包起来，在最外层先判断登录状态，这就是 AppLayout 组件的功能了。

为 AppLayout 组件添加如下代码：

```js
...

import React, { useContext, useEffect } from 'react';
import { getCookie } from 'utils/utils'
import { AppContext } from 'context-manager.js'

const AppLayout = () => {

  const { state, dispatch } = useContext(AppContext)

  useEffect(() => {
    var x_openid = getCookie('token')
    if(x_openid){
      // 这里加入具体的用户系统逻辑判断，比如发送 x_openid 到后台用 cookie-parser 解密，再查找该 openid 是否真实有效，若是则成功登录
      dispatch({type: "LOGIN_SUCCESS"})
    }
  }, [])

...

```

Demo 简化了登录的判断，如果有 cookie 就直接登录了。

验证一下，访问 `/about`，直接显示了页面而没有跳转到登录页，成功。

## 4. 总结

至此，微信一键登录功能看似已经做完了，但其实还存在一个页面跳转的『bug』，下一篇具体讲这个问题，以及做出合理的优化。