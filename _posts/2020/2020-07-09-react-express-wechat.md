---
layout: post
title: React + Express 搭起简易微信登录框架（一）
tags: [react, wechat]
---

## 0. 引言

刚开始学习 [React](https://reactjs.org/)，做了一个简单的 Demo 实现微信一键登录。

因为微信接口必须写在后台（appid, secret 不能暴露到客户端），所以用 [Express](http://expressjs.com/) 简单搭了一个服务端，页面跳转用 [react-router](https://reactrouter.com/)，状态管理使用 v16.8 推出的 Hooks API（useReducer 和 useContext）。

因为后端服务器跨域的问题，开发环境按照官方的推荐使用 [http-proxy-middleware](https://create-react-app.dev/docs/proxying-api-requests-in-development)。生产部署的时候需要额外配置一下 nginx，静态资源和后端请求分开提供服务。

虽然是个小 Demo，但是写起来东西还是挺多的，于是分开几篇逐步介绍几块内容。本文主要是关于项目初始化和登录状态的管理与跳转。

## 1. 初始化项目

直接使用官方工具 [create-react-app](https://create-react-app.dev/docs/getting-started) 创建项目，然后安装一下依赖包：

* react-router-dom
* node-sass
* axios
* express
* http-proxy-middleware

为了方便组件的导入，更改 package.json 文件的 scripts 代码块，增加 `NODE_PATH=src` 变量如下：

```json
"scripts": {
  "start": "NODE_PATH=src react-scripts start",
  "build": "NODE_PATH=src react-scripts build",
  "test": "NODE_PATH=src react-scripts test",
  "eject": "react-scripts eject"
},
```

这样就可以简化以后的导入命令：

```js
import LoginPage from '../../pages/Login'
--->
import LoginPage from 'pages/Login'
```

项目目录结构如下：

```
project
|-- src
    |-- pages
        |-- AppLayout.js
        |-- Login.js
        |-- NoWechat.js
    |-- components
    |-- route
        |-- AuthorizedRoute.js
    |-- images
    |-- utils
    |-- stylesheets
        | -- pages
            |-- _appLayout.scss
            |-- _login.scss
        | -- components
        | -- variables
        | -- application.scss
    |-- App.js
    |-- context-manager.js
    |-- setupProxy.js
|
|-- app_server.js
```

我习惯于集中管理样式文件，所以把所有统一放到 stylesheets 文件夹下，不过本文主要记录功能实现，就不罗列样式代码了。

接下来看具体实现步骤，分别介绍以上文件的内容及作用。

## 2. 状态管理

使用 useReducer Hook 来记录用户的登录状态，并用 useContext Hook 向内部组件穿透。

状态管理本身抽到 `src/context-manager.js` 文件中，任何状态的扩展都在这个文件中进行。先来看一下它的内容：

```js
import React from 'react'

export const AppContext = React.createContext()

export const initState = {
  isLogin: false,
}

export const reducer = (state, action) => {
  switch(action.type) {
    case "LOGIN_SUCCESS":

      return {
        isLogin: true,
      }

    default:
      return state
  }
}
```

然后只需导入该文件，通过 `<AppContext.Provider value=>` 将状态穿透到所有内部组件即可。

任何内部组件要拿到 state 只需像这样申明：

```js
import { AppContext } from 'context-manager.js'
const { state } = useContext(AppContext)
```

后面会看到具体用法。

## 3. 登录跳转

这个 Demo 的主要功能是用户登录，它的前端界面的核心功能是根据用户身份（状态）显示不同页面，合法则显示受保护页面，不合法则显示登录页。

在这个登录跳转的功能上我参照了 [react-router 的官方例子](https://reactrouter.com/web/example/auth-workflow)，用一个自定义路由组件（用于验证）把受保护页面路由包裹起来。

下面是 App.js:

```js
import React, { useReducer } from 'react'
import { BrowserRouter as Router } from 'react-router-dom'
import 'stylesheets/application.scss'
import AppLayout from 'pages/AppLayout'
import { AppContext, reducer, initState } from 'context-manager' // 状态步骤穿透 1

function App() {
  const [state, dispatch] = useReducer(reducer, initState)  // 状态步骤穿透 2

  return (
    <AppContext.Provider value={{state, dispatch}}>  // 状态步骤穿透 3
      <Router>
        <AppLayout />          
      </Router>
    </AppContext.Provider>
  )
}

export default App;
```

下一步就是 AppLayout 组件，包裹了所有的页面：

```js
import React from 'react';
import { Route, Switch } from 'react-router-dom'

import AuthorizedRoute from 'route/AuthorizedRoute'
import LoginPage from 'pages/Login'

const { Content, Footer } = Layout

const AppLayout = () => {

  return (
    <Switch>
      <Route path="/login">
        <LoginPage />
      </Route>
      
      <AuthorizedRoute path="/">
        <Route exact path="/">
          <h1>This is Home Page.</h1> 
        </Route>
        <Route path="/about">
          <h1>This is About Page.</h1> 
        </Route>
      </AuthorizedRoute>
    </Switch>
  )
}

export default AppLayout
```

然后是关键的 AuthorizedRoute 组件：

```js
import React, { useContext } from 'react';
import { Route, Redirect, useLocation } from 'react-router-dom'
import { AppContext } from 'context-manager.js'      // 状态引用步骤 1

const AuthorizedRoute = ({ children, ...rest }) => {

  const { state } = useContext(AppContext)     // 状态引用步骤 2
  const location = useLocation()

  return (
    <Route
      {...rest}
      render={({ location }) =>
        state.isLogin ? (              // 通过 isLogin 状态判断，显示访问页面 or 显示登录页面
          children
        ) : (
          <Redirect
            to={
              {
                pathname: '/login',
                state: { from: location }  // 将访问页面 location 传给 login，这样登录后就能跳转
              }
            }
          />
        )
      }
    />
  )
}

export default AuthorizedRoute
```

最后是 Login 组件：

```js
import React, { useContext } from 'react'
import { Redirect, useLocation } from 'react-router-dom'
import { AppContext } from 'context-manager.js'

const LoginPage = () => {

  const { state, dispatch } = useContext(AppContext)
  const location = useLocation()
  const { from } = location.state || { from: { pathname: '/' }}

  if(state.isLogin) {
    return (
      <Redirect to={from.pathname} />    // 如果直接访问登录页，则跳转到 '/'
    )
  } else {
    return (
      <button onClick={() => dispatch({type: "LOGIN_SUCCESS"})}>
        微信一键登录
      </button>
    )
  }
}

export default LoginPage
```

这里简单地实现了登录功能，点击『微信一键登录』按钮触发在 `context-manager.js` 中定义好的 dispatch 方法，更改状态 isLogin 为 true，同时触发 Login 组件的 render，然后 Redirect 到访问页面，登录成功。

下一步就是改写登录的业务逻辑，即实现微信登录。但是，如引言所说，微信登录接口不能放在客户端调用，我们不能做一个纯前端应用然后让用户在微信端直接一键登录，因为这样会暴露敏感数据。所以在实现微信登录之前，下一篇会先介绍如何搭建一个最简化的 Express 后台。

---

### 参考资料：

* [React 官网](https://reactjs.org/)
* [React Router 官网 ](https://reactrouter.com/web/guides/quick-start)
* [create-react-app](https://create-react-app.dev/docs/getting-started)
* [How to Structure Your React Project](https://daveceddia.com/react-project-structure/)