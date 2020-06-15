---
layout: post
title: React Router 学习笔记
tags: [react]
---

## 一、三大基本对象

### 1. history

来自 React Router 的两个主要依赖包之一 [https://github.com/ReactTraining/history/blob/master/docs/getting-started.md](https://github.com/ReactTraining/history/blob/master/docs/getting-started.md)

有如下属性和方法：

* length - (number) history stack 中的条目数量
* action - (string) 当前动作（PUSH, REPLACE, POP）
* location - (object) 当前 location，包含属性：pathname、search、hash、state
* push(path, [state]) - (function) push 新的条目到 history stack
* replace(path, [state]) - (function) replace 当前的条目
* go(n) - (function) 在 history stack 中移动 n 个条目
* goBack() - (function) 等于 go(-1)
* goForward() - (function) 等于 go(1)
* block(prompt) - (function) 组织跳转

不要使用 history.location 因为 history 是易变的（mutable）

### 2. location

结构：

```jsx
{
  key: "ac3df4",
  pathname: "/somewhere",
  search: "?some=search-string",
  hash: "#howdy",
  state: {
    key: "value"
  }
}
```

location 可以传入两个组件：Route 和 Switch，用来替换当前页面的真实 location

### 3. match

包含了 `<Route path>` 组件匹配 URL 的信息，属性如下：

* params - (object) key/value 对，来自 URL 的动态路劲部分
* isExact - (bool) 完全匹配整个 URL
* path - (string) 用来匹配的 path 模式内容，对嵌套 `<Route>` 很有用
* url - (string) 匹配的 URL 部分，对嵌套 `<Link>` 很有用

***[null match](https://reacttraining.com/react-router/web/api/match/null-matches）的机制还不理解***

## 二、Hooks

* useHistory      - 获取 history 对象
* useLocation     - 获取 location 对象
* useParams       - 获取 route url 的动态 params
* useRouteMatch   - 获取 match 对象

## 三、组件

### 1. Router 组件

底层接口，一般使用更高阶的包裹组件：BrowerRouter、HashRouter、MemoryRouter、NativeRouter、StaticRouter

```jsx
<Router
  history={customHistory}
>
```

### 2. BrowerRouter 组件

使用 HTML5 的 history API 来保持 UI 与 URL 的同步

```jsx
<BrowserRouter
  basename={optionalString}    // string: 类似 namespace，比如："/my_app"，之后的 url 都会接在其后
  getUserConfirmation={optionalFunc}  // func: 默认使用 window.confirm
  forceRefresh={optionalBool}  // bool: 跳转时强制整页刷新
  keyLength={optinalNumber}    // number: location.key 的长度，默认是 6
>
```

*注：只接受一个子元素*

### 3. Link 组件

创建一个 html a 元素用来完成导航（to 有三种写法：string、object、func）

```jsx
<Link
  // string
  to="/url?key=value"
  // location object
  to={
    {
      pathname: "/url",
      search: "?key=value",
      hash: "#the-hash",
      state: { fromDashboard: true }
    }
  }
  // func, 接受 current location 作为参数，返回 object || string
  to={location => ({...location, pathname: "/url"})}
  to={location => `${location.pathname}?key=value`}
  // 自定义内容，当然也可以直接用子元素
  component={FancyLink}
>
```

### 4. NavLink 组件

特殊的 Link，用来识别并更新当前 Link

```jsx
<NavLink
  to={}    // 用法同 Link
  activeClassName="selected"
  activeStyle={{
    color: "red"
  }}
  exact    // 完全匹配
  strict   // 匹配 URL 尾部的斜杠 "/"
  // 自定义函数匹配，比如不仅仅像 activeClasssName 那样验证 pathname
  isActive={(match, location) => {
    if(!match) return false

    const eventID = parseInt(match.params.eventID)
    return !isNaN(eventID) && eventID % 2 === 1
  }}
  // 替换 isActive 函数中的 location 参数对象，默认是当前页面的 location
  location={anotherLocation}
>
```

### 5. Route 组件

用来匹配当前路由并渲染元素

```jsx
<Route
  path="/url/:params"        // 不提供 path 属性则始终匹配
  location={anotherLocation} // 替换当前 location，但是会被（如果设置） Swich 组件的 location 覆盖
  exact
  strict
  sensitive
>
```

官方推荐的用来渲染内容的方式是用子元素，即：

```jsx
ReactDOM.render(
  <Router>
    <div>
      <Route exact path="/">
        <Home />
      </Route>
      <Route path="/news">
        <NewsFeed />
      </Route>
    </div>
  </Router>,
  node
);
```

Route 同时还支持另外几种渲染方式，主要是为了兼容在 hooks 出现之前的旧版本

* `<Route component>`
* `<Route render>`
* `<Route children>`

三种方式都会接收到三个同样的 route props（三大基本对象）:

* match
* location
* history

#### ——> component

component 的用法很直观，把定义好的组件名直接传给 `<Route component>` 属性即可。

官方例子：

```jsx
import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router, Route } from "react-router-dom";

// All route props (match, location and history) are available to User
function User(props) {
  return <h1>Hello {props.match.params.username}!</h1>;
}

ReactDOM.render(
  <Router>
    <Route path="/user/:username" component={User} />
  </Router>,
  node
);
```

需要注意的是，router 使用 `React.createElement` 方法为该 component 创建一个新的 React 元素，意味着如果给 `<Route component>` 传一个内联函数，则每个 render 都会触发一个新 component 的创建。这会导致已存在的组件的 umounting 和新组件的 mounting，而不是仅仅更新组件。

所以，如果要使用内联函数，则使用 `render` 或者 `children` prop（下面介绍）

#### ——> render：func

`<Route render>` 方式方便了内联函数的使用，解决了上述不期望的 unmounting 动作。

官方例子

```jsx
import React from "react";
import ReactDOM from "react-dom";
import { BrowserRouter as Router, Route } from "react-router-dom";

// convenient inline rendering
ReactDOM.render(
  <Router>
    <Route path="/home" render={() => <div>Home</div>} />
  </Router>,
  node
);
```

***注：<Route component> 优先于 <Route render>，两者不能同时启用***

#### ——> children：func

`<Route children>` 用法与 render 一样，唯一的不同是无论路由是否匹配 children 的函数都会执行，即内容都会渲染。

如下官方例子中，进入 "/" 页面同时会渲染 "/somewhere" 和 "/somewhere-else" 两个组件，并且 match 是 null。

官方例子

```jsx
import React from "react";
import ReactDOM from "react-dom";
import {
  BrowserRouter as Router,
  Link,
  Route
} from "react-router-dom";

function ListItemLink({ to, ...rest }) {
  return (
    <Route
      path={to}
      children={({ match }) => (
        <li className={match ? "active" : ""}>
          <Link to={to} {...rest} />
        </li>
      )}
    />
  );
}

ReactDOM.render(
  <Router>
    <ul>
      <ListItemLink to="/somewhere" />
      <ListItemLink to="/somewhere-else" />
    </ul>
  </Router>,
  node
);
```

### 6. Redirect 组件

重定向到新的 location，会在 history stack 中覆盖当前的 location（同 HTTP 3xx）

```jsx
<Redirect
  to={}       // string || object, 用法同 Link
  push        // pushState instead of replaceState
  from="/url" // 仅在 <Switch> 中可用，用来匹配路由
  exact       // 和 from 搭配使用，用来精准匹配
  strict      // 和 from 搭配使用，用来匹配末尾斜杠 "/"
  sensitive   // 大小写敏感
>
```

### 7. Switch 组件

渲染第一个匹配当前 location 的 `<Route>` 或 `<Redirect>` 子组件

Switch 的特点是只会渲染子组件中匹配的第一个 Route，不会渲染多个。而纯 Route 组件数组会渲染所有匹配的组件。

```jsx
<Switch
  location={anotherLocation} // 替换当前 location
>
```

Switch 组件的子元素只能是 Route 或 Redirect