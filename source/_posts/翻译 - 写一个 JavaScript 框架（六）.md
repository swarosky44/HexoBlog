---
title: 写一个 JavaScript 框架（六）
date: 2017-11-22 16：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是教你写一个 JavaScript 框架系列文章的第六篇。在这个章节中，我们将讨论强大的自定义元素以及它在现代前端框架中所扮演的角色。

这个系列包括以下章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. [数据绑定简介](http://swarosky44.github.io/2017/09/15/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%E6%A1%86%E6%9E%B6%EF%BC%88%E5%9B%9B%EF%BC%89/)
5. [用 ES6 Proxy 实现数据绑定](http://swarosky44.github.io/2017/11/17/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%BA%94%EF%BC%89/)
6. 自定义元素（当前章节）
7. 客户端路由

<!-- more -->

## 组件时代
组件在近几年已经完全统治了 web。几乎所有的现代前端框架 - 列如 React、Vue 或者 Polymer - 都在利用组件来实现模块化。他们虽然提供着各不相同的 API，在各自的引擎下工作。但是它们以及其他现代框架在以下几个特性是相同的：

- 他们都有一个 API 用于定义组件并且通过命名或者选择器完成注册。
- 他们都提供生命周期的钩子函数，用于建立组件内部的逻辑，根据状态同步视图。

上述的特性都错过了一个简易的原生 API，直到最近，随着自定义元素的规范最终确定下来，这一情况也将有所改变。自定义元素可以取代上述的特性，但是它也不总是最完美的选择，我们来一起探究下原因。

## 自定义元素
自定义元素是[Web 组件标准](https://www.w3.org/standards/techs/components#w3c_all)的一部分，2011 年时自定义元素就已经出现，在最终确定下来之前，它一直都有两份不同的规范。最终确定下来的版本更像是一个简单的基于组件构建的框架，而不是框架作者手中的一个工具。它提供了一个高层次的 API 用于定义组件，但是它缺少了许多无法替补的新特性。

如果你对自定义元素不够了解，请在开始之前先阅览一下这篇[文章](https://developers.google.com/web/fundamentals/getting-started/primers/customelements)。

### 自定义元素 API
自定义元素 API 是基于 ES6 的类。元素可以继承自原生的 HTML 元素或者其他自定义元素，并且它们可以扩展新的属性和方法。它们也可以重写在规范中定义的一系列方法，作为钩子函数连接到元素的生命周期中。

```JavaScript
class MyElement extends HTMLElement {
  // these are standard hooks, called on certain events
  constructor() { ... }
  connectedCallback () { ... }
  disconnectedCallback () { ... }
  adoptedCallback () { ... }
  attributeChangedCallback (attrName, oldVal, newVal) { ... }

  // these are custom methods and properties
  get myProp () { ... }
  set myProp () { ... }
  myMethod () { ... }
}

// this registers the Custom Element
customElements.define('my-element', MyElement)
```

定义之后，元素能够以具体名称在 HTML 或 JS 代码中实例化。

```HTML
<my-element></my-element>
```

基于类的 API 是十分简洁的，但是在我的观点中，它缺乏灵活性。作为一个框架作者，我更加倾向于已经废弃的 V0 版本 [API](https://www.html5rocks.com/en/tutorials/webcomponents/customelements/)，它是基于老式的原型链语法。

```JavaScript
const MyElementProto = Object.create(HTMLElement.prototype)

// native hooks
MyElementProto.attachedCallback = ...
MyElementProto.detachedCallback = ...

// custom properties and methods
MyElementProto.myMethod = ...

document.registerElement('my-element', { prototype: MyElementProto })
```

虽然它没有那么优雅，但是它可以再 ES6 以及 ES6 之前的代码完美兼容。在另一方面，将 ES6 类的语法和一些 ES6 之前的语法混合在一起，会让代码变得非常复杂。

举个例子，我需要能够控制组件从 HTML 接口继承的能力。ES6 的类使用关键字 `extends` 实现继承，并且需要开发人员键入 `MyClass extends ChosenHTMLInterface`。

这样的做法对我而言不够理想，因为 NX 是基于中间件而不是类。在 NX 中，这个接口可以设置 `element` 的属性，它接受一个可用的 HTML 元素名称 - 列如 `button`。

```JavaScript
nx.component({ element: 'button' })
  .register('my-button')
```
为了实现这个，我不得不以原型系统为基础去仿造一个 ES6 的类。简单来说，它比想象中要难实现，需要的 ES6 的 `Reflect.construct` 以及对性能不友好的 `Object.setPrototypeOf` 函数。

```JavaScript
function MyElement () {
  return Reflect.construct(HTMLElement, [], MyElement)
}
const myProto = MyElement.prototype
Object.setPrototypeOf(myProto, HTMLElement.prototype)
Object.setPrototypeOf(MyElement, HTMLElement)
myProto.connectedCallback = ...
myProto.disconnectedCallback = ...
customElements.define('my-element', MyElement)
```

这是在某次非常偶然的情况下我发现 ES6 的类很笨拙。我认为它们对于日常工作是够用的了，但是当我想要发挥一门语言的全部力量时，我更倾向于使用原型链继承。

### 生命周期钩子
自定义元素有五个生命周期的钩子函数，它们按照特定的事件依次触发。

- `constructor` 是一个元素的实例化。
- `connectedCallback` 是当一个元素挂载到 DOM 树上时。
- `disconnectedCallback` 是当一个元素脱离 DOM 树时。
- `adoptedCallback` 是当一个元素通过 `importNode` 或者 `cloneNode` 进入一个新的文档时。
- `attributeChangedCallback` 是当一个元素被观察的属性发生变化时。

`constructor` 和 `connectedCallback` 适合建立组件内部的状态和逻辑，而 `attributeChangedCallback` 用于映射组件由 HTML 属性构成的特性，反之亦然。
`disconnectedCallback` 适合在组件的实例被清理之后调用。

当这些钩子函数组合在一起，可以提供一组非常棒的功能，不过我目前还欠缺了 `beforeDisconnected` 和 `childrenChanged` 的回调。`beforeDisconnected` 钩子函数用于实现组件离开前的动画十分有用，不过如果没有对 DOM 的大量修补和重载的话，这不太可能实现。

`childrenChanged` 的钩子函数用于连接状态和视图是必不可缺的。如下例：

```JavaScript
nx.component()
  .use((elem, state) => state.name = 'World')
  .register('my-element')
```
```JavaScript
<my-component>
  <p>Hello: ${name}!</p>
</my-component>
```

这是一个很简单的模板，它将 `name` 属性注入到视图中。如果用户想要用其他什么元素来取代这个 `p` 元素，框架就必须去提示一下这个变化。它必须清除之前的 `p` 元素并且将新元素注入。`childrenChanged` 可能并不会作为一个开发者接口暴露出来，但是对于一个框架，了解一个组件内容的变化是非常有必要的。

就像我提到的，自定义元素缺少 `childrenChanged` 的回调，但是它可以通过以前的 [MutationObserver API](https://developer.mozilla.org/en/docs/Web/API/MutationObserver) 来实现。 MutationObservers 也可以提供可供选择的 `connectedCallback`、`disconnectedCallback` 以及 `attributeChangedCallback` 钩子函数给老版的浏览器。

```JavaScript
// create an observer instance
const observer = new MutationObserver(onMutations)

function onMutations (mutations) {
  for (let mutation of mutations) {
    // handle mutation.addedNodes, mutation.removedNodes, mutation.attributeName and mutation.oldValue here
  }
}

// listen for attribute and child mutations on `MyComponentInstance` and all of its ancestors
observer.observe(MyComponentInstance, {
  attributes: true,
  childList: true,
  subtree: true
})
```

如果抛开自定义元素的简易 API 不谈，上例可能会让人对自定义元素是否真的有必要存在疑问。
在下一小节中，我将会支出 `MutationObservers` 和 自定义元素之间的差异，并讨论什么时候选择使用哪个。

## 自定义元素 VS MutationObservers
自定义元素的回调函数是在 DOM 发生转变时同步调用的，而 `MutationObservers` 收集转变并分批的异步地调用回调函数。在创建逻辑上这并不会有什么问题，但是在清除实例的时候会引起一些意想不到的 bug。当数据已经处理过后仍然会存在一小段时间是非常危险的行为。

另一个很重要的差异是，`MutationObservers` 不会捕获影子 DOM。监听影子 DOM 内部的转变需要自定义元素或者添加一个 `MutationObservers` 到影子 DOM 的根组件上。如果你从未了解过影子 DOM，可以看[这里](https://developers.google.com/web/fundamentals/getting-started/primers/shadowdom)获得更多的了解。

最后，在钩子函数上他们也有一点差异，自定义元素有 `adoptedCallback` 钩子函数，而 `MutationObservers` 可以在任意深度上去监听文本或者子元素的转变。

考虑到以上，将两者的精华之处结合起来会是一个很不错的做法。

## 结合自定义元素与 MutationObservers
因为自定义元素[尚未被广泛支持](http://caniuse.com/#search=custom%20elements)，`MutationObservers` 必须用来观察 DOM 的转变。这里有两个观点用于使用它们。

- 在顶层 API 上使用自定义元素，并且用 `MutationObservers` 实现替代方案。
- 建立 `MutationObservers` 的 API 并且当自定义元素可用时使用自定义元素以获得一些提升。

我选择使用后者的建议，即使浏览器能够全面支持自定义元素，我也会用 `MutationObservers` 来观察子元素的变化。

下一个版本的 NX 架构中我将会为老式浏览器添加一个 `MutationObservers`。尽管如此，在现代浏览器中，它会为大部分顶层组件提供自定义元素的钩子，同时也会在 `connectedCallback` 中添加一个 `MutationObservers`。这个 `MutationObservers`将用于观察组件深层次的转变。

它只会在文档流内部观察变化，而文档流是在框架的控制下。响应函数大致如下：

```JavaScript
function registerRoot (name) {
  if ('customElements' in window) {
    registerRootV1(name)
  } else if ('registerElement' in document) {
    registerRootV0(name)
  } else {
     // add a MutationObserver to the document
  }
}

function registerRootV1 (name) {
  function RootElement () {
    return Reflect.construct(HTMLElement, [], RootElement)
  }
  const proto = RootElement.prototype
  Object.setPrototypeOf(proto, HTMLElement.prototype)
  Object.setPrototypeOf(RootElement, HTMLElement)
  proto.connectedCallback = connectedCallback
  proto.disconnectedCallback = disconnectedCallback
  customElements.define(name, RootElement)
}

function registerRootV0 (name) {
  const proto = Object.create(HTMLElement)
  proto.attachedCallback = connectedCallback
  proto.detachedCallback = disconnectedCallback
  document.registerElement(name, { prototype: proto })
}

function connectedCallback (elem) {
  // add a MutationObserver to the root element
}

function disconnectedCallback (elem) {
// remove the MutationObserver from the root element
}
```

它为现代浏览器提供了更好的性能表现，因为它们只需要去处理最小的一组 DOM 变化。

