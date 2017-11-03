---
title: 写一个 JavaScript 框架（四）
date: 2017-09-15 14：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是教你写一个 JavaScript 框架系列文章的第四篇。在这个章节中，我将会讲解脏检查、访问器中的数据绑定技术以及它们各自的优劣。

这个系列包括以下章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. 数据绑定简介（当前章节）
5. 用 ES6 Proxy 实现数据绑定
6. 自定义元素
7. 客户端路由

<!-- more -->

## 数据绑定简介

> 数据绑定是一种很常用的技术用于绑定和同步数据源的生产者和消费者

这是一种通用的定义，指出了通常数据绑定技术需要构建的模块。

  - 定义生产者和消费者的句柄
  - 定义触发同步操作的变量的句柄
  - 定义监听生产者变量改变的方法
  - 定义当改变发生时的同步函数的句柄。文本从现在开始这个函数我们统称为 `handler()`

在不同的数据绑定技术中会用不同的方法实现以上的模块，在下一个章节中就会介绍两个这样的技术：脏检查 & 访问器。它们都各有优缺点，在介绍完之后，我会简单的评价一下两种技术。

## 脏检查
脏检查可能是当下最被人们熟悉的数据绑定技术。在核心概念上它十分简单，不需要用到复杂的语法技巧，这让它在以前是一个不错的选择。

### 语法
定义生产者和消费者不需要特别的语法，直接用 JavaScript 的空对象即可。

```JavaScript
const provider = {
  message: 'Hello World',
};
const consumer = document.createElement('p');
```

同步通常是由于生产者的属性转变所触发。这些需要被观察转变的属性必须明确映射到它们的 `handler()` 上。

```JavaScript
observe(provider, 'message', message => {
  consumer.innerHTML = message;
});
```

这个 `observer()` 函数仅仅是保存 `(provider, property) -> handler` 这个映射用于之后使用。

```JavaScript
function observer(provider, prop, handler) {
  provider._handlers[prop] = handler;
};
```

有了这个，我们的句柄可以用于定义生产者和消费者，并且给属性的转变，注册 `handler()` 函数。这个开放的 API 在我们库里就已经准备好了，下一步就是内部的实现。

## 监听转变

脏检查被称为脏是有一个原因的，它用周期性的检测替代了直接监听属性的转变。从现在开始，我们现在称这个检查为消化系统。消化系统迭代循环每个由 `observe()` 添加进来的 `(provider, property) -> handler`，并且检测属性值是否从上一次迭代过程中发生了转变。一个简单的实现如下：

```JavaScript
function digest() {
  providers.forEach(digestProvider);
}

function digestProvider(provider) {
  for (let prop in provider._handlers) {
    if (provider._preValues[prop] !== provider[prop]) {
      provider._preValues[prop] = provider[prop];
      handler(provider[prop]);
    }
  }
}
```

这个 `digest()` 函数需要一次次的运行用来确保状态的同步。

## 访问器技术

访问器技术是目前的一个趋势。它没有那么大范围的支持是因为它需要 ES5 getter/setter 函数，但是它让数据绑定的实现变得十分优雅。

## 语法

定义生产者需要特别的语法。生产者的空对象必须传递给 `observable()` 函数，将它转变成一个可观察的对象。

```JavaScript
const provider = observable({
  greeting: 'Hello',
  subject: 'World',
});
const consumer = document.createElement('p');
```

比起通过 `handler()` 映射的语法，访问器的语法的不便要小多了。在脏检查中，我们不得不明确地定义每个需要被观察属性，如下：

```JavaScript
observe(provider, 'greeting', greeting => {
  consumer.innerHTML = greeting + ' ' + provider.subject;
});
observe(provide, 'subject', subject => {
  consumer.innerHTML = provider.greeting + ' ' _ subject;
});
```

这样的语法是冗余且粗糙的。访问器技术可以自动地发现属性转变，并且调用对应的 `handler()` 函数，这让我们可以写出十分简洁的代码：

```JavaScript
observe(() => {
  consumer.innerHTML = provider.greeting + ' ' + provider.subject;
});
```

`observe()` 的实现和脏检查的实现是不一样的。它仅仅执行一下传递进来的 `handler()` 函数并且标记为当前活跃状态。

```JavaScript
let activeHandler;

function observe(handler) {
  activeHandler = handler;
  handler();
  activeHandler = undefined;
}
```

需要注意的是，这里我们通过使用 `activeHandler` 变量和 JavaScript 的单线程的特性去保持执行 `handler()` 函数。

### 监听转变

这是访问器技术的来源。生产者通过增加 `getter/setter` 在背后给了我们巨大的支持。它通过以下的方式拦截对生产者属性的 `get/set` 操作：

- get: 如果当前有个 `activeHandler` 正在运行，保存 `(provider, property) -> activeHandler` 映射在之后使用。
- set: 运行所有与该属性映射的 `handler()` 函数。

![](http://blog-assets.risingstack.com/2016/Sep/The_accessor_data_binding_technique-1473151244710.svg)

下面的代码示范了对生产者的一个属性的简单实践：

```JavaScript
function observableProp(provider, prop) {
  const value = provider[prop];
  Object.defineProperty(provider, prop, {
    get() {
      if (activeHandler) {
        provider._handlers[prop] = activeHandler;
      }
      return value;
    },
    set(newValue) {
      value = newValue;
      const handler = obj._handlers[prop];
      if (handler) {
        activeHandler = handler;
        handler();
        activeHandler = undefined;
      }
    }
  })
}
```

在上一节中提到的 `observable()` 函数以递归方式遍历生产者的属性，并将其全部转换成被观察的通过上面提及的 `observableProp` 函数。

```JavaScript
function observable(provider) {
  for (let prop in provider) {
    observableProp(provider, prop);
    if (typeof provider[prop] === 'object') {
      observable(provider[prop]);
    }
  }
}
```

这是一个十分简单的实现，但是用于比较两种技术是已经足够了的。

## 两种技术的比较

在这个章节中，我会简明地指出脏检查和访问器技术之间的优劣。

### 语法
脏检查无需特别的语法来定义生产者和消费者，但是在映射 `(provider, property) -> handler()` 时候是繁冗和笨重的。

访问器技术需要生产者被 `observable()` 包裹，但是会自动将 `handler()` 完成映射。对于大型项目的数据绑定，这是一个必须要有的特性。

### 表现

脏检查的性能是总所周知的差劲。在每次的消化循环中它必须多次检查每个 `(provider, property) -> handler`。除此之外，即使当前的 App 是处于空闲状态，它也必须保持工作，因为它无法知道何时属性会发生转变。

在性能上访问器技术则更快，但是对于大型的可观察的对象，性能会有所下降。将生产者的每个属性都转换，通常来说是一种过度使用。一种解决办法是在需要时动态地构建 `getter/setter`，而不是一次性全部构建完成。除此之外，还有一种办法实在不需要处理的属性上再包裹一层 `noObserve()` 函数用于告诉 `observable()` 远离这个属性，这将额外引入一些新的语法进来。

### 灵活性

脏检查自然地就会为动态添加的自定义属性以及访问器上的属性工作。

访问器技术在这一方面则不如人意，自定义属性并不能被支持，因为它们不在 `getter/setter` 树上。比如数组上就会发生这种问题，但是它可以通过再添加新属性之后手动执行 `observableProp()` 来修复。`Getter/Setter` 属性也不被支持，因为访问器不能再被访问器包裹。一种常用的变通方法是用 `computed()` 函数取代 `getter`。这也会引入更多的自定义语法。

### 选择的时机

脏检查在这方面并没有给我们太多的自由，由于我们实际上无法知道属性发生转变的时机。`handler()` 函数只能异步执行，通过一次又一次循环运行 `digest()`。

通过访问器技术添加的 `Getters/Setters` 是同步触发的，因此我们有选择的自由。我们还可以选择是立即执行 `handler()` 或者保存在异步操作之后的批处理中。前者让我们有可预测的优势，后者让我们减少重复的操作以提升性能。

## 关于下一篇文章

在下一篇文中，我将会介绍 [nx-observe](https://github.com/RisingStack/nx-observe) 数据绑定的库并且解释如何用 ES6 Proxies 取代 ES5 getters/setters 以清除更多的访问器技术的缺陷。

> 翻译自 [Writing a JavaScript Framework - Introduction to Data Binding, beyond Dirty Checking](https://blog.risingstack.com/writing-a-javascript-framework-data-binding-dirty-checking/)
