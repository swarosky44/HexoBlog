---
title: 写一个 JavaScript 框架（一）
date: 2017-07-26 10：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

最近看到一个非常棒的系列文章，作者是 NX 框架的作者，主要是讲在开发 NX 的过程中的一些关键问题的解决方法。加上最近读 Vue 的模板计算十分痛苦，于是我愉快的决定 ~ 不看了。先把这个系列的文章看完 🙃

这是系列的第一篇 -- 项目结构。

<!-- more -->

就职于 RisingStack 的 JavaScript 工程师 Bertalan Miklos 在过去的几个月里写了一个下一代客户端框架，叫作 [NX](http://nx-nxframework.rhcloud.com)。在写一个 JavaScript 框架的系列文章里，Bertalan 分享了在开发过程中他的一些心得：

在这个章节中，我会说明 NX 是如何架构的以及我是如何解决可扩展性、依赖注入以及私有变量这些方面的难题。

这个系列文章还有以下章节：
1. 项目结构（当前章节）
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. 数据绑定简介
5. 用 ES6 Proxy 实现数据绑定
6. 自定义元素
7. 客户端路由

## 项目结构
尽管世界上并不存在能够适用于所有项目的结构，但是仍然存在一些通用的准则。对于这些感兴趣的的同学可以参阅 [Nodejs 项目结构指南](https://blog.risingstack.com/node-hero-node-js-project-structure-tutorial/) 系列文章。

### NX 框架梗概
NX 目标是成为一个开源社区驱动的项目，这样方便推广和扩展。
- 它有现代框架的全部特性。
- 它没有外在的依赖，除了一些 polyfills。
- 它仅仅由 3000 行左右的代码组成。
- 没有任何一个模块超过 300 行。
- 没有任何一个特性模块需要超过 3 个依赖。

它最终的关系图如下：
![](http://blog-assets.risingstack.com/2016/Jul/javascript_framework_in_2016_the_nx_project_structure-1467708098586.svg)

这个结构为开发框架过程中的一些典型难题提供了一种解决方法：
- 可扩展性
- 依赖注入
- 私有变量

### 实现可扩展性
对于社区驱动的项目，易于扩展是必须的。为了实现这个目标，项目应有一个小巧的内核以及一个预定义的依赖处理系统。前者能够保证项目是清晰易懂的，而后者能够保证项目在扩展过程中始终保持清晰易懂。

在这个模块，我们将聚焦于实现一个小巧的内核。

现代框架一个主要特色是能够创建自定义元素并且和普通 DOM 元素一样使用。NX 有一个独立函数 `component` 作为它的核心，而这个独立函数就是为了实现这个特色。它允许用户去注册和配置一个新的元素类型。

```JavaScript
component(config).register('comp-name');
```
这个被注册的空白元素`comp-name`能够在 DOM 中被实例化。
```HTML
<comp-name></comp-name>
```
下一步是确保元素能够实现其他的一些新特性，同时保持自身的简洁以及可扩展性，依赖注入就擅长于让这些新特性不会影响到核心函数。

### 用中间件实现依赖注入（DI）
如果你不是很熟悉DI，我推荐你先阅读一下[Nodejs 中的依赖注入](https://blog.risingstack.com/dependency-injection-in-node-js)。
> DI是一种用于实现将一个或多个依赖（或服务）通过引用或者传递的方式注入到依赖对象的设计模式。

DI 解决了非常麻烦的依赖管理，但是也产生了一个新的问题。用户必须了解如何去设置和注入全部依赖，现代客户端框架绝大多数都是提供了 DI 容器来代替用户的工作。
> 一个 DI 容器是一个知道如何去实例化并且配置注入依赖的对象。

另一种方法是中间件 DI 模式，它被广泛用于服务端（Express、Koa）。它的诀窍在于所有注入的依赖都有相同的接口，可以用相同的方式注入。这样，就不需要一个 DI 容器了。我用的就是这种方式，如果你曾经用过 Express，那么下面的代码你将会非常熟悉。

```JavaScript
component()
  .use(paint)
  .use(resize)
  .register('comp-name');

function paint(elem, state, next) {
  elem.style.color = 'red';
  next();
}

function resize(elem, state, next) {
  elem.style.width = '100px';
  next();
}
```

当组件挂载到 DOM 时中间件会执行，并且将新特性作为特性扩展到组件上。通过不同的库来扩展组件一个相同的特性，将会导致命名冲突。暴露私有变量会加深这个问题并且可能造成其他人的一些误用。

只提供一小部分的公共 API 并且隐藏其他部分是避免上述问题的最佳实践。

### 处理私有变量
在 JavaScript 中我们使用函数作用域来实现私有性。跨作用域访问私有属性时，我们会在它们之前加上 '_' 去标识它们的私有性，然后暴露它。这可以避免一些误用的情况，但依然避免不了命名冲突。一个更好的方案是使用 ES6 的 Symbol。

> Symbol 一种唯一且不可变的数据类型，它可以用作对象属性的键。
（btw：这篇文章下面评论里，有人指出作者对于 Symbol 的解释有误导，的确在本文情景中 Symbol 可以避免误用以及命名冲突，但是 Symbol 数据类型并不是私有属性，因为其他人依然可以用 Object.getOwnPropertySymbols() 来访问一个对象所有 Symbol 类型的键。这就不满足私有属性的定义了。)

下面的代码演示了如何使用 Symbol。

```JavaScript
const color = Symbol();
function colorize(elem, state, next) {
  elem[color] = 'red';
  next();
}
```

现在 `red` 只能通过拥有对 color 声明的 Symbol 才能访问（以及 DOM 元素）。`red` 属性值的私密性可以通过对 `color` symbol 的不同暴露程度来控制。设置合理数量的私有变量，用一个中心对象来存储 symbol 变量，这种做法是相当优雅的。

```JavaScript
exports.private = {
  color: Symbol('color from colorize'),
};
exports.public = {};
```
在 Index.js 中如下：
```JavaScript
const symbols = require('./symbols');
exports.symbols = symbols.public;
```

存储对象在项目的所有模块中都是可访问的，但是私有模块就不会对外暴露。公共模块用于暴露一些低层次的特性给开发者。这样杜绝了误用的情况，因为开发者们必须清楚的声明自己所需的 symbol 才能够使用它。更甚者，symbol 属性名是不可覆盖的，不像字符串命名，因此命名冲突也就不存在了。

下面总结几种情景：
#### 1. 公共变量
正常使用
```JavaScript
function (elem, state, text) {
  elem.publicText = 'Hello World!';
  next();
}
```

#### 2. 私有变量
跨作用域变量中的私有变量，应该有一个 symbol 类型的键，并且添加到私有的注册对象中。

```JavaScript
exports.private = {
  text: Symbol('private text'),
};
exports.public = {};
```

当有需求访问变量时：
```JavaScript
const private = require('symbols').private;
function (elem, state, next) {
  elem[private.text] = 'Hello World！'；
  next();
}
```

#### 3.半私密变量
底层的 API 应该有 Symbol 类型的键，并且注册到公开的注册对象中。
```JavaScript
exports.private = {
  text: Symbol('private text'),
};
exports.public = {
  text: Symbol('exposed text'),
};
```

当有需求访问变量时

```JavaScript
const exposed = require('symbols').public;
function(elem, state, next) {
  elem[exposed.text] = 'Hello World!';
  next();
}
```

## 总结
如果你对 NX 框架感兴趣，可以参阅[主页](http://nx-nxframework.rhcloud.com)。热衷于探索的读者可以在[GitHub Repo](https://github.com/RisingStack/nx-framework)参阅源码。

我希望你能认为这一篇好的文章，下次将会讨论调度执行。

BTW：这里读私有属性的一块读得晕晕乎乎的，感觉不论怎么写，都不符合私有属性的认知。在 Java 中定义一个私有属性通过修饰符 `private` 实现，仅有自身类的作用域可以访问，如果需要向外暴露，只能通过提供 getter 和 setter。这里使用 Symbol 确实可以有效避免命名冲突，但是避免误用方面，给我的感觉是多加了几层阻挠，让开发者想清楚当前正在访问什么级别的变量而已。

AnyWay, 他说可以就可以 ~ 🤔

> [原文](https://blog.risingstack.com/writing-a-javascript-framework-project-structuring/)
