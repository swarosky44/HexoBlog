---
title: 写一个 JavaScript 框架（五）
date: 2017-11-17 16：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是教你写一个 JavaScript 框架系列文章的第五篇。在这个章节中，我将会讲解如何使用 ES6 的 Proxies 来实现一个简洁高效的数据绑定库。

这个系列包括以下章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. [调度执行](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. [数据绑定简介](http://swarosky44.github.io/2017/09/15/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%E6%A1%86%E6%9E%B6%EF%BC%88%E5%9B%9B%EF%BC%89/)
5. 用 ES6 Proxy 实现数据绑定（当前章节）
6. 自定义元素
7. 客户端路由

<!-- more -->

## 预备知识

尽管 ES6 使 JavaScript 变得十分优雅，但是大部分的新语法都只是语法糖。Proxies 是少有的非补充性的新增语法之一。如果你还不熟悉这个语法，请在此之前简单速读一下 [MDN Proxy docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)。

对 ES6 的 Reflect API 以及 Set、Map、WeakMap 对象有简单的了解也有一定的帮助。

## nx-observe 库

[nx-observe](https://github.com/RisingStack/nx-observe) 是一个代码量少于 140 行的数据绑定解决方案。它暴露出 `observeable(obj)` 和 `observe(fn)` 两个函数，它们用于创建被观察者和观察函数。当被观察者对象的属性发生改变的时候，观察函数会自动执行。例子如下：

```javascript
// this is an observable object
const person = observable({name: 'John', age: 20})

function print () {
  console.log(`${person.name}, ${person.age}`)
}

// this creates an observer function
// outputs 'John, 20' to the console
observe(print)

// outputs 'Dave, 20' to the console
setTimeout(() => person.name = 'Dave', 100)

// outputs 'Dave, 22' to the console
setTimeout(() => person.age = 22, 200)
```
`print` 函数作为参数传递进 `observe()` 里面，每当 `person.name` 或者 `person.age` 发生改变时就会执行。`print` 就被称为一个观察函数。
如果你想看更多的案例，请翻阅 [GitHub readme](https://github.com/RisingStack/nx-observe#example) 或者 [NX home page](http://nx-framework.com/docs/spa/observer)，那里有更多的案例。

## 实现一个简单的被观察者

在这个小节里，我将会解释在 `nx-observe` 内部的逻辑。首先，我会展示观察者是如何发现被观察者的属性发生改变的。然后我会解释这些改变是如何触发的观察函数。

### 注册变化
变化是通过使用 ES6 Proxies 包裹被观察者完成注册的。在 [Reflection API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect) 的帮助下 proxies 可以无缝地拦截所有 `get` 和 `set` 操作。

下面代码中用到的变量 `currentObserver` 和 `queueObserver`, 将会在下一小节中解释清楚。现在只需要明白 `currentObserver` 总是会指向当前正在执行的观察函数，`queueObserver` 是一个使观察函数按照队列顺序执行的函数。

```JavaScript
/* maps observable properties to a Set of
observer functions, which use the property */
const observers = new WeakMap()

/* points to the currently running
observer function, can be undefined */
let currentObserver

/* transforms an object into an observable
by wrapping it into a proxy, it also adds a blank
Map for property-observer pairs to be saved later */
function observable (obj) {
  observers.set(obj, new Map())
  return new Proxy(obj, {get, set})
}

/* this trap intercepts get operations,
it does nothing if no observer is executing
at the moment */
function get (target, key, receiver) {
  const result = Reflect.get(target, key, receiver)
   if (currentObserver) {
     registerObserver(target, key, currentObserver)
   }
  return result
}

/* if an observer function is running currently,
this function pairs the observer function
with the currently fetched observable property
and saves them into the observers Map */
function registerObserver (target, key, observer) {
  let observersForKey = observers.get(target).get(key)
  if (!observersForKey) {
    observersForKey = new Set()
    observers.get(target).set(key, observersForKey)
  }
  observersForKey.add(observer)
}

/* this trap intercepts set operations,
it queues every observer associated with the
currently set property to be executed later */
function set (target, key, value, receiver) {
  const observersForKey = observers.get(target).get(key)
  if (observersForKey) {
    observersForKey.forEach(queueObserver)
  }
  return Reflect.set(target, key, value, receiver)
}
```

如果 `currentObserver` 尚未赋值 `get` 的代理将不会执行任何事情。反之，它会匹配观察者的属性与当前正在执行的观察函数，并且将他们保存到 `observers` 的映射中。观察者们会以 `Set` 集合保存在每个被观察属性中。这样可以避免发生冲突。

`set` 的代理会检索所有和变化属性匹配的观察者，然后按照队列顺序执行他们。

你可以绘制一个下面这样的图表，然后一步步解释清楚 `nx-observe` 的案例代码是怎么运作的。

![](https://blog-assets.risingstack.com/2016/11/writing-a-javascript-framework-data-binding-with-es6-proxy-observables-code.png)

1. 创建被观察者 `person`
2. `print` 设置为 `currentObserver`
3. `print` 开始执行
4. `person.name` 在 `print` 中被访问
5. `person` 的 `get` 代理触发
6. 和 `(person, name)` 匹配的观察者可以通过 `observers.get(person).get('name')` 检索得到
7. `print` 保存到 Set 集合中
8. 当访问了 `person.age` 时，重复执行 4- 7 步骤
9. `${person.name}, ${pseron.age}` 在控制台上完成打印
10. `print` 执行完成
11. `currentObserver` 被赋空值
12. 其他代码开始运行
13. `person.age` 被赋值为 22
14. `set` 的代理触发
15. 和 `(person, age)` 匹配的观察者的 Set 集合通过 `observers.get(person).get('age')` 被检索到
16. 在 Set 集合中的观察函数队列顺序执行
17. `print` 函数再次执行

## 观察者的运作
分批异步的调用队列内的观察者，可以让系统有更好的性能。在注册观察者时，观察者们是被异步地添加到 `Set` 中。一个 `Set` 集合不允许包含相同的对象，因此，多次添加同一个观察者是不会多次触发的。如果一个 `Set` 集合是空的，那么遍历和执行观察者队列中的观察函数会延迟执行。

```javascript
  /* contains the triggered observer functions,
  which should run soon */
  const queuedObservers = new Set()

  /* points to the currently running observer,
  it can be undefined */
  let currentObserver

  /* the exposed observe function */
  function observe (fn) {
    queueObserver(fn)
  }

  /* adds the observer to the queue and
  ensures that the queue will be executed soon */
  function queueObserver (observer) {
    if (queuedObservers.size === 0) {
      Promise.resolve().then(runObservers)
    }
    queuedObservers.add(observer)
  }

  /* runs the queued observers,
  currentObserver is set to undefined in the end */
  function runObservers () {
    try {
      queuedObservers.forEach(runObserver)
    } finally {
      currentObserver = undefined
      queuedObservers.clear()
    }
  }

  /* sets the global currentObserver to observer,
  then executes it */
  function runObserver (observer)
    currentObserver = observer
    observer()
  }
```
上述的代码证明了无论何时只要有一个观察者在运行，就会有一个全局变量 `currentObserver` 是指向它的。当前 `currentObserver` 运行时能够监听所有被观察者的属性变化，我们给 `currentObeerver` 开启 get 的代理。
## 创建一个动态的被观察者树

到目前为止，我们的模型在面对简单的单一的数据结构的时候表现是很不错的，但是需要我们手动去将每一个对象的新属性值包装成可观察的。下例中的代码就不会按照预期工作：

```javascript
const person = observable({data: {name: 'John'}})

function print () {
  console.log(person.data.name)
}

// outputs 'John' to the console
observe(print)

// does nothing
setTimeout(() => person.data.name = 'Dave', 100)
```
为了让代码能够正常运作，我们必须将 `observable({ data: { name: 'John' } })` 替换成 `observable({ data: observable({ name: 'John' }) })`。
不过我们可以通过小小修改一下 `get` 的代理来解决这个麻烦。

```javascript
function get (target, key, receiver) {
  const result = Reflect.get(target, key, receiver)
  if (currentObserver) {
    registerObserver(target, key, currentObserver)
    if (typeof result === 'object') {
      const observableResult = observable(result)
      Reflect.set(target, key, observableResult, receiver)
      return observableResult
    }
  }
  return result
}
```

上述代码中的 `get` 代理在先将返回值传入 `observable()` 中完成代理再返回的 - 如果返回值是对象类型。从性能的角度上来看这样是完美的，因为被观察者只有在他们真正需要被观察的时候才会创造出来。

## 和 ES5 的技术对比
用 ES6 的 `Proxies` 可以实现一个与用 ES5 属性访问器(getter/setter) 十分相似的数据绑定技术。许多流行库都采用这种技术，例如[MobX](https://mobxjs.github.io/mobx/)和[Vue](https://vuejs.org/)。使用代理而不是属性访问器有两个主要的优势和一个严重的劣势。

### 自定义属性
自定义属性是 JavaScript 中动态添加的属性。ES5 的技术并不支持自定义属性因为访问器为了保证能够拦截每个操作，必须预先确定每个属性。这是一个技术原因，这也是为什么现在创建一个预先定义好键值的数据中心日渐流行起来的原因。

另一方面，Proxy 技术是能够支持自定义属性的，因为它定义的是对象，它拦截对象每个属性的每个操作。

一个典型的使用自定义属性的例子是使用数组。JavaScript 的数组如果没有添加或者删除元素的能力将会变得非常无用。ES5 的技术解决这个问题通常是提供自定义的或者重写 `Array` 方法。

### Getters 和 Setters
ES5 的库通过一些特殊的语法为绑定的属性们提供了计算的功能。这些功能它们是有原生的实现，叫作 getters 和 setters。但是因为 ES5 的库在内部使用 getter/setter 来实现数据绑定的逻辑，因为不能用来实现属性访问器的功能。

Proxy 拦截属性的每次访问和改变，包括 getters 和 setters，因此这对于 ES6 而言不会造成问题。

### 劣势
使用 Proxies 的最大劣势是浏览器的支持度问题，目前仅仅是现代浏览器才会有支持，并且大部分的 Proxy API 都没有支持脚本。

## 一点建议
虽然介绍的数据绑定库已经是一个可用的了，但是为了易于理解我是做了一些简化的。从下面的建议中你可以发现一些由于简化而遗漏的观点。

### 清理
内存泄漏是很恶心的。因为使用了 `WeakMap` 来保存观察者，现在介绍的代码从某种意义上是避免了这个问题的。这意味着观察者是与被观察者和被观察的垃圾是联系在一起的。

尽管如此，一个可用的案例是一个伴随着频繁变动的 DOM 的耐用的数据中心。在这个例子中，DOM 应该在被当做垃圾收集起来之前释放所有它注册了的观察者。这个功能在上例子中遗失了，但是你可以通过查阅[nx-observe code](https://github.com/RisingStack/nx-observe/blob/master/observer.js)看看 `unobserve()` 函数是怎么实现的。

### 多次代理
Proxies 是透明的，意味着这里并没有原生的方法去判断一个对象是 Proxy 代理对象还是一个空对象，他们甚至可以是无限相似的。因此如果没有必要的预备措施的话，我们可能会将一个被观察者一次又一次的代理。

这里有许多巧妙的办法将一个 Proxy 对象和普通对象区分开。在上例中，一个可用的方法是使用 `WeakSet` 来记录已被代理的对象，之后检查是否包含对象来判断是否已经做过代理了。如果你对 nx-observe 是如何实现 `isObservable()` 函数感兴趣的话可以查阅[code](https://github.com/RisingStack/nx-observe/blob/master/observer.js)。

### 继承
nx-observe 对原型链继承同样有效。下例证明了这个有什么意义：

```javascript
const parent = observable({greeting: 'Hello'})
const child = observable({subject: 'World!'})
Object.setPrototypeOf(child, parent)

function print () {
  console.log(`${child.greeting} ${child.subject}`)
}

// outputs 'Hello World!' to the console
observe(print)

// outputs 'Hello There!' to the console
setTimeout(() => child.subject = 'There!')

// outputs 'Hey There!' to the console
setTimeout(() => parent.greeting = 'Hey', 100)

// outputs 'Look There!' to the console
setTimeout(() => child.greeting = 'Look', 200)
```
`get` 代理操作会涉及原型链上的每一个节点，直到找到指定属性。因为观察者被注册到了每一个它可能被需要的地方。

有一些边缘情况是由一个鲜为人知的事实引起的，即 `set` 代理操作也会在原型链中运行(偷偷地)，但是这里不会涉及到这些情况。

### 内部属性
代理还可以拦截内部属性的访问。你的代码可能会用到许多你通常意想不到的内部属性。有一些键名是一些[众所周知的符号](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)。这些属性也能被 Proxies 正确的代理，不过有一些地方是有 bug。

### 异步
观察者可以在 `set` 操作拦截后同步的运作。这有一些好处：比如低复杂度、可控的时间和更好的堆栈跟踪，但是在某些场景下也会造成混乱。

试想在单次循环中，将 1000 个元素推入到一个被观察的数组中。数组的长度会变化 1000 次，与之关联的观察者也会快速连续执行 1000 次。这意味着运行一组 1000 个功能的相同的函数，这是件相当无用的事情。

另一个有问题的场景是双向观察。下面的代码如果是同步运行则会造成无限循序：

```javascript
const observable1 = observable({prop: 'value1'})
const observable2 = observable({prop: 'value2'})

observe(() => observable1.prop = observable2.prop)
observe(() => observable2.prop = observable1.prop)
```

基于上述原因 nx-observe 排列观察者是没有重复的，并且让它们作为微任务分批执行以避免[FOUC](https://en.wikipedia.org/wiki/Flash_of_unstyled_content)。如果你对微任务的概念不熟悉，可以查阅之前的[文章](http://swarosky44.github.io/2017/07/31/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6/#more)
