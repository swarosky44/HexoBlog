---
title: 写一个 JavaScript 框架（七）
date: 2017-12-05 16：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是写一个 JavaScript 框架系列文章的最后一章节。在这个章节中，我将会讨论 JavaScript 的客户端路由和服务端路由有什么差异及其原因。

这个系列包括以下章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. [数据绑定简介](http://swarosky44.github.io/2017/09/15/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%E6%A1%86%E6%9E%B6%EF%BC%88%E5%9B%9B%EF%BC%89/)
5. [用 ES6 Proxy 实现数据绑定](http://swarosky44.github.io/2017/11/17/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%BA%94%EF%BC%89/)
6. [自定义元素](http://swarosky44.github.io/2017/11/22/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E5%85%AD%EF%BC%89/)
7. 客户端路由（当前章节）

<!-- more -->

## web 上的路由
网页要么是由服务端输出，要么由客户端输出，或者两者混合使用输出，不论如何，有一定复杂度的网页都必须能够处理路由。

对于服务端而言，路由的处理是由后端负责。传统项目中 URL 改变或者参数改变时候会输出一个新的页面。不过，网页应用通常会需要保存当前用户的状态，这对于需要输出无数个页面的服务端而言十分困难。

通过预先取出整个应用然后选择适合的页面，客户端框架解决了上述问题。前端路由实践起来和后端十分相似。唯一的不同是它从客户端获取资源而不是服务端。在本文，我将会解释为什么他们二者还是需要做一些差异处理。

### 启蒙于后端的路由

大部分的前端路由库都是从服务端获得的启发。

当 URL 改变时，它们都会选择合适的路由处理器来决定和输出所需的组件。它们的架构和后端的十分相似，唯一的差异是处理器函数所做的事情。

要想证明它们的相似性，我们可以发现服务端框架 [Express](https://expressjs.com/en/guide/routing.html)、客户端框架 [page.js](https://visionmedia.github.io/page.js/) 以及 [React](https://github.com/ReactTraining/react-router) 在路由上的语法十分相似。

```JavaScript
// Express
app.get('/login', sendLoginPage)
app.get('/app/:user/:account', sendApp)
```

```JavaScript
// page.js
page('/login', renderLoginPage)
page('/app/:user/:account', renderApp)
```

```JavaScript
<!-- React -->
<Router>
  <Route path="/login" component={Login}/>
  <Route path="/app/:user/:account" component={App}/>
</Router>
```

React 在 JSX 之后还隐藏了一些逻辑，但是它们所做的事情是一样的，在介绍动态参数之前，它们所做的都很完美。

在上述例子中，单个用户可以有多个账号，并且当前的账号可以自由切换。如果在 `App` 页面上账号切换了，相关的处理器会为新账户重新选择并输出一个 `App` 组件 - 当然，在现有的组件上更新一些数据可能也够用了。

这对于基于 VDOM 的解决方案而言并不是什么大问题 - 因为他们会计算 DOM 并且只更新需要更新的地方 - 但是对于传统框架，它还可能意味着大量的不必要的工作。

### 处理动态参数
我想避免的是当 URL 参数改变时就重新输出整个页面。为了解决这个问题，我首先将动态参数从路由中剥离了出去。

在 NX 中，路由决定了当前哪个组件或者视图用于展现，进入哪个 URL 的路径下。动态参数控制当前页面所需要展示的数据，并且它们始终是在问号后面的参数里。

这意味着 `/app/:user/:account` 将会变形成 `/app?user=userId&account=accountId`。这样有一点冗余不过还是很整洁的，并且这样允许我将客户端路由分割成页面路由和参数路由。前者导航 `App Shell`，后者导航 `Data Shell`。

### App Shell

你可能已经熟悉了 [App Shell 模型](https://developers.google.com/web/fundamentals/architecture/app-shell)，它由于 Google 推出的 PWA 而流行起来。

>App Shell 是支持用户界面所需的最小的 HTML、CSS 和 JavaScript。

在 NX 中，页面路由负责在 App Shell 中导航。一个简单的路由结构如下：

```JavaScript
<router-comp>
  <h2 route="login"/>Login page</h2>
  <h2 route="app"/>The app</h2>
</router-comp>
```

它和前面的例子很相似，尤其是 React。但是这里有一个重要的差异。它不处理 `user` 和 `account` 参数，相反，它只负责导航 App Shell。

这样只需要处理一个静态树遍历的问题了。路由树基于 URL 的路径名开始遍历，所展现的组件取决于最终所找到的节点。

![](https://blog-assets.risingstack.com/2017/03/client-side-routing-javascript-path-routing-diagram.svg)

上述图片解释了 `/settings/profile` URL 是如何最终决定输出的视图。下面是代码：

```JavaScript
nx.components.router()
  .register('router-comp')
```

```JavaScript
<a iref="home">Home</a>
<a iref="settings">Settings</a>
<router-comp>
  <h2 route="home" default-route>Home page</h2>
  <div route="settings">
    <h2>Settings page</h2>
    <a iref="./profile">Profile</a>
    <a iref="./privacy">Privacy</a>
    <router-comp>
      <h3 route="profile" default-route>Profile settings</h3>
      <h3 route="privacy">Privacy settings</h3>
    </router-comp>
  </div>
</router-comp>
```

这个例子向我们展示了一个具有默认路由以及相对路由的嵌套路由结构。如你所见，它简单到只需要配置 HTML 并且它就可以像大部分的文件系统一样工作起来。你可以用绝对路径（`home`）或者相对路径（`./privacy`）的链接实现导航。路由代码表现如下：

![](https://blog-assets.risingstack.com/2017/03/client-side-routing-javascript-path-routing-example.gif)

这个简单的结构同样可以用来创造更加强大的路由。多个路由树同时工作时的平行路由就是例子之一。[NX docs page](https://nx-framework.com/docs/start)的侧边栏菜单和主页内容就是以这种方式在工作。它有两个并行的嵌套路由，可以同步改变侧边栏菜单以及主页内容。

### Data Shell

和 App Shell 不一样，Data Shell 是个还没有宣传过的理念。实际上，它也只是我一个人在用。Data Shell 是指驱动数据流动的动态参数数据池。他不会切换当前页面，仅仅只是改变页面的数据。改变当前页面通常也会改变数据池，但是改变数据池内的数据并不会造成页面的重新渲染。

通常情况下，Data Shell 是由当前页面的初始值组成的集合，代表了当前页面的状态。因此它可以用来保存、加载或者分享状态。为了实现这点，它必须能够映射 URL、本地数据库或者浏览器历史记录等能够让它全局化的数据。

NX 的 `control` 组件以及其他组件可以利用钩子函数向数据池子中注入一个已经声明了的配置对象，这个对象决定了参数如何与组件状态、URL、历史记录以及网页数据库进行交互。

```JavaScript
nx.components.control({
  template: require('./view.html'),
  params: {
    name: { history: true, url: true, default: 'World' }
  }
}).register('greeting-comp')
```

```javascript
<p>Name: <input type="text" name="name" bind/></p>
<p>Hello @{name}</p>
```

上例创建了一个组件，并且让 `name` 属性和 URL 、浏览器历史记录同步起来。具体表现如下：

![](https://blog-assets.risingstack.com/2017/03/client-side-routing-javascript-param_routing.gif)

要感谢 [ES6 Proxy 的透明反应性]，它让同步能够时刻保持连续性。你可以写一些简单的 JavaScript 代码试下，两者会在后台互相同步起来。下面的示意图给出了一个宏观的示意：

![](https://blog-assets.risingstack.com/2017/03/client-side-routing-javascript-parameter_routing-diagram.svg)

这种简洁的声明式语法鼓励开发人员们在编码之前花上几分钟更好的设计和整合一下页面。并非所有的参数都适合加入 URL 或者当改变时添加一条新的历史记录。这里有许多种各不相同的使用方式，每一个都应该被合理的设计。

- 一个文本过滤参数应该是 `url` 参数，因为它可能需要分享给其他用户。

- 当前账户的 ID 应该是 `url` 参数以及 `history` 参数，因为当前账户 ID 应该是可分享的，并且它可能会被切换掉，因此需要添加一条历史记录。

- 用户的视觉偏好应该是 `durable` 参数（保存在本地数据库中）因为它对每个人都应该被持久保存，但是又不应该分享给他人。

这些仅仅一些可能出现的情况，几分钟的小小设计可以让参数在你面对的情境中更加适用。

### 结合起来

路径路由和参数路由对彼此都是相互独立的，但是他们被设计成结合在一起可以更好的工作。路径路由在 App Shell 中负责页面导航，然后参数路由负责掌管控制页面的状态和 Data Shell。

页面间的参数池可能会存在差异，所以有一个显示的 API 可以用 JavaScript 和 HTML 来改变当前页面。

```html
<a iref="newPage" $iref-params="{ newParam: 'value' }"></a>
```

```javascript
comp.$route({
  to: 'newPage',
  params: { newParam: 'value' }
})
```

在上例中，NX 会自动添加 `active` 的 CSS 类名到链接上，你可以通过 `options` 配置对象来配置所有链接的普通属性，列如继承过来的属性以及链接事件。

查阅 [routing 文档](https://www.nx-framework.com/docs/middlewares/route) 可以了解更多特性。

## Demo

下例展示了路由参数与响应式数据系统的结合。在 NX 应用里它完全可以正常运作。复制代码然后粘贴到一份空的 HTML 文件中，在现代浏览器中打开文件，体验一下吧。

```Javascript
<script src="https://www.nx-framework.com/downloads/nx-beta.2.0.0.js"></script>

<script>
nx.components.app({
  params: {
    title: { history: true, url: true, default: 'Gladiator' }
  }
}).use(setup).register('movie-plotter')

function setup (comp, state) {
  comp.$observe(() => {
    fetch('http://www.omdbapi.com/?r=json&t=' + state.title)
      .then(response => response.json())
      .then(data => state.plot = data.Plot || 'No plot found')
  })
}
</script>

<movie-plotter>
  <h2>Movie plotter</h2>
  <p>Title: <input type="text" name="title" bind /></p>
  <p>Plot: @{plot}</p>
</movie-plotter>
```

title 属性自动同步到了 URL 和 浏览器历史记录中。这个功能通过 `comp.$observe` 的订阅，当 title 属性改变时，可以自动的获取电影的相关介绍。这个例子创建了一个强大的数据响应系统，并且在浏览器中能够完美运行。

![](https://blog-assets.risingstack.com/2017/03/client-side-routing-javascript-example-app-2.gif)

这个应用并没有展示路径路由。查阅[intro app](https://github.com/nx-js/intro-example)、[NX Hacker News clone](https://github.com/nx-js/hackernews-example) 或者 [路径路由](https://www.nx-framework.com/docs/middlewares/route) 和 [参数路由](https://www.nx-framework.com/docs/middlewares/params) 的文档可以了解更多复杂的案例。

## 总结

这一段不是翻译，写一个 JavaScript 框架的系列文章翻译到这一步就没了。全篇翻译下来，相当吃力，可以很确信的说这些翻译中存在大量语义不通顺或者不明确的问题，之后会花费一定时间多次修改，好好打磨一下。英语果然是限制大部分国内程序员技术水平的天然障碍啊 🙃 。

在此之后，还会多多联系翻译能力，不仅仅局限于 Web 前端，Android、IOS 文章都会多多涉猎。最近也一直有在玩 Android, 会尝试写几篇文章记录一下学习历程以及心得。仅此。
