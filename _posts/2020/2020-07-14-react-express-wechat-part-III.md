---
layout: post
title: React + Express 搭起简易微信登录框架（三）
tags: [react, express, wechat]
---

## 0. 引言

上两篇已经完成所有准备工作：项目初始化、登录验证与跳转、后台 Express 以及服务代理。

本文将集成微信的 API，完整流程：用户在手机微信客户端点击『微信一键登录』按钮，然后 Express 发起微信 API 请求，最后返回结果到前端并成功登录。

***[Demo 的完整代码在 [这里](https://github.com/SpiderEvgn/react-wechat-login-demo)]***

## 1. 微信接口

先简要介绍一下微信登录的接口，详情查看[微信官方文档](https://developers.weixin.qq.com/doc/offiaccount/Getting_Started/Overview.html)。

首先，需要创建一个测试号，微信提供了个人测试号的机制方便用户测试相关功能：[接口测试号申请](https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Requesting_an_API_Test_Account.html)

同时，为了测试方便，再去下载微信提供的[开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)，这样就可以在电脑上模拟微信客户端测试了。

通过微信客户端授权（获取用户 openid 和信息）的方式称为『[网页授权](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html)』，网页授权有两种基本模式：『snsapi_base』和『snsapi_userinfo』，前者无需用户授权，即在用户无感知的情况下即可获取用户 openid，后者需要用户显式授权（跳出用户同意界面），授权后即可获取用户的微信个人信息（昵称、头像、地区等）。

Demo 使用的是『snsapi_base』的方式，只获取 openid。另一种操作方式几乎完全一样，就不赘述了。

具体的授权流程[官网](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html)介绍得非常详细，简要来说就两步：

1. 请求 code
2. 用 code 换 openid 和 access_token（Demo 中没用到 access_token）

如果是『snsapi_userinfo』接口，需要获取个人信息，那么还有第三步：

* 用 access_token 和 openid 请求个人信息

## 2. 请求 code

先来看微信授权的第一步，拼接一个访问地址（[官网案例](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html#0)）：

```
https://open.weixin.qq.com/connect/oauth2/authorize?appid=wx520c15f417810387&redirect_uri=https%3A%2F%2Fchong.qq.com%2Fphp%2Findex.php%3Fd%3D%26c%3DwxAdapter%26m%3DmobileDeal%26showwxpaytitle%3D1%26vb2ctag%3D4_2030_5_1194_60&response_type=code&scope=snsapi_base&state=123#wechat_redirect
```

注意，这里的回调域名 `redirect_uri` 需要到公众号后台配置，Demo 在测试号中配置：

![uri-config](/assets/img/posts/2020/react-wechat/uri-config.jpg "URI config")

测试号允许配置 ip 地址，可以直接配置本机的局域网地址，比如笔者的配置如下：

![uri-config-2](/assets/img/posts/2020/react-wechat/uri-config-2.jpg "URI config 2")

然后，我们去 Login 组件拼接这个 URI 字符串，并赋给登录按钮，这样当我们点击『微信一键登录』的时候就会访问这个地址，微信收到请求后就会带上 code 回调我们自己的页面，我们就拿到 code 可以进行下一步了。

在 Login 组件加入如下配置：

```js
const wechat_config = {
  appid: '[your-appid]',
  domain: '192.168.0.101%3A3000',
  scope: 'snsapi_base',
  state: 'test'
}
const wechat_oauth2_link = `https://open.weixin.qq.com/connect/oauth2/authorize?appid=${wechat_config.appid}&redirect_uri=http%3A%2F%2F${wechat_config.domain}%2F&response_type=code&scope=${wechat_config.scope}&state=${wechat_config.state}&connect_redirect=1#wechat_redirect`
```

同时替换 `button` 的 `onClick` 事件：

```js
onClick={() => window.location.href = wechat_oauth2_link}
```

完成后当我们再次点击『微信一键登录』按钮的时候，实际我们是访问了拼接的 URI，微信收到请求后会回调我们提供的地址并带上 code，不过由于没有登录，所以页面上的最终显示仍然会从根页面跳转到登录界面。但其实如果你暂时去除保护路由，可以观察到浏览器的地址栏是如下形式：

```
http://192.168.0.101:3000/?code=0617SU730pFWuI1bv7b30PJV7307SU7c&state=test
```

成功拿到 code，下一步就是请求 openid。

## 3. 前端转发 code 到后台 Express

回调地址是根路由，我抽象到了包裹它的 AuthorizedRoute 组件中完成 openid 的请求。

区分微信回调和普通的访问的关键就是 code 参数，所以需要先判断一下 URI 请求参数是否带有 code。

给 AuthorizedRoute 组件添加如下代码：

```js
// 引入 useLocation
import { Route, Redirect, useLocation } from 'react-router-dom'

...

const AuthorizedRoute = ({ children, ...rest }) => {

  // 获取 dispatch，用来在拿到 openid 后更新登录状态
  const { state, dispatch } = useContext(AppContext)

  const location = useLocation()

  // 截取 code 参数
  var code = ''
  if(location.search.split('&')[0].substr(1, 4)==='code'){
    code = location.search.split('&')[0].substr(6)
  }

  ...
```

然后在 useEffect 中判断 code 是否存在，若肯定则发起 openid 请求。改写 useEffect 方法如下：

```js
useEffect(() => {
  // 判断请求 url 是否来自微信回调，即用户通过微信授权登录
  if(code){
    axios(`/api/fetch-wechat-userinfo?code=${code}`)
      .then(res => {
        if(res.data.status === 'ok'){
          dispatch({type: "LOGIN_SUCCESS"})
          console.log(res.data)
        }else{
          // 错误处理
        }
      })
      .catch(error => {
        // 错误处理
      })
  }
}, [])
```

将 code 转发到后台服务，通过 Express 在后台发起真正的微信 API 请求去获取 openid，返回成功结果后用 dispatch 方法更新登录状态（可以记录 openid 测试一下）。

## 4. 请求 openid

最后来到真正执行微信 API 的地方：app_server.js

改写原来的方法如下：

```js
// 别忘了导入 axios 库
const axios = require('axios')

app.get('/api/fetch-wechat-userinfo', async (req, res) => {
  var openid = ''

  const wechat_config ={
    appid: '[your-appid]',
    secret: '[your-secret]'
  }
  const wechat_openid_link = `https://api.weixin.qq.com/sns/oauth2/access_token?appid=${wechat_config.appid}&secret=${wechat_config.secret}&code=${req.query.code}&grant_type=authorization_code`

  await axios(wechat_openid_link)
    .then(async res => { 
      openid = res.data.openid
    })
    .catch(err => {
      // 错误处理
      openid = 'error'
    })

  return res.json({
    status: 'ok',
    openid: openid
  })
})
```

这里使用 async/await 同步 axios 的 http 请求，因为我们要获取到 openid 才能回给前端。

重启 app_server.js，再次点击『微信一键登录』，就能在控制台看到输出的 openid 了（当然你也可以把 openid 保存到状态然后打印在页面上）。

## 5. 总结

至此，通过微信客户端网页授权的方式一键登录的功能就实现了。

但是作为登录功能来说，还缺一个很重要的步骤：状态保持。

下一篇我将加入 cookie 来记住登录状态。