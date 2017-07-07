---
title: Vue - 响应式数据构造过程
date: 2017-06-30 15：30
tags:
  - Vue
  - JavaScript
categories: Vue
---

其实关于 Vue 如何构造响应式数据以及指令、模板解析过程等等分析的文章已经很多了。这里自己写一次是因为，我TM源码读了几次了，每次过一两个星期就忘得一干二净了 🙄

<!-- more -->

 # 预备知识

个人感受，最好先准备一下相关知识点，能轻松一点。 直接干撸源码，会有种人生终极三问的迷茫感 ...（当然水平高的话就不一定了。）

- MVVM 的简单实现：[240行实现一个简单的框架](https://zhuanlan.zhihu.com/p/24475845)
- 观察者模式：[图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html) & [我的文章](http://swarosky44.github.io/2017/05/02/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/#more)
- Object 各种元属性操作：[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object)

# 脑图

![](http://7xro5v.com1.z0.glb.clouddn.com/Vue_Observer1.png)

脑图是用来帮助理解的，整个响应式对象的构造过程简单说来就三点：

- 递归处理数据结构；
- 所有原始数据类型的值都将经过 defineReactive 处理，完成 getter 和 setter 的劫持。
- 所有数组都将重定义数组部分常用方法。

# defineReactive

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  // TODO
}
```

首先我们可以看一下这个函数调用的几个参数

@param {Object} obj [需要构造的对象]
@param {String} key [键]
@param {Any} val [值]
@param {Function} customSetter [setter 调用时注入的回调函数]

```javascript
const dep = new Dep()
```

这个 dep 是 Dep 类的实例，用于存储 & 收集 `obj[key]` 这个属性值的所有观察者。

```javascript
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
  return
}
// cater for pre-defined getter/setters
const getter = property && property.get
const setter = property && property.set
```

这段代码用于提取 `obj` 的元属性。 如果元属性中 `configurable` 已经被定义为 `false`，那么该对象 `obj` 意味着是不可写入的。所以无法构造成响应式。反之，则提取出元属性中定义的 `getter` 和 `setter`，这样是为了在后面重新定义 `obj` 的元属性时，不会丢失之前可能是用户定义的 `getter` 和 `setter`；

```javascript
let childOb = observe(val)
```

dang！！！这里是重点中的重点，不过暂时我们先不要管，直接跳过看下面的就好。

```Javascript
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    // TODO
  },
  set: function reactiveSetter (newVal) {
    // TODO
  },
})
```

这里就开始 & 完成了对 `obj[key]` 的整个劫持过程。
重新定义了 `obj[key]` 的一些元属性，最主要的就是这里的 `getter` 和 `setter` 了。

```javascript
function reactiveGetter () {
  const value = getter ? getter.call(obj) : val
  // 只有在 Vue 内部依赖收集过程中才会触发
  if (Dep.target) {
    // 当前 dep 收集 Dep.target 观察者
    dep.depend()
    if (childOb) {
      // childOb.dep 收集 Dep.target 观察者
      childOb.dep.depend()
    }
    if (Array.isArray(value)) {
      // value 数组中每个值都会完成 dep 对 Dep.target 的收集
      dependArray(value)
    }
  }
  return value
}
```

1. 首先我们执行了之前提取出来的 `getter`，获取了返回值，这里的意图很简单，我其实不关心用于访问这个 `obj[key]` 返回什么，因为用户可能会自定义一个 `getter` 或者使用默认 `getter`。我们要做的只是保持用户的意图，然后正常返回这个值。

2. 第二部比较抽象，这里其实不太方便扩展解释，因为这里又会提到许多新东西，比如 Dep 这个观察者容器类 & Watcher 观察者构造类。我们只需要明白的是，这是 Vue 进行内部依赖收集过程中才会触发。而 `dep.depend()` 执行的内容就是让当前的 dep 收集当前 Dep 类的静态属性 target。而 `obj[key]` 的值如果是对象或者数组，则也需要构造成响应式，所以也需要收集当前 Dep.target 这个依赖。

BTW:
- Dep.target 在 flow 的限制下，只能是 Watcher 类的实例。

```javascript
function reactiveSetter (newVal) {
  const value = getter ? getter.call(obj) : val
  /* eslint-disable no-self-compare */
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  /* eslint-enable no-self-compare */
  if (process.env.NODE_ENV !== 'production' && customSetter) {
    customSetter()
  }
  if (setter) {
    setter.call(obj, newVal)
  } else {
    val = newVal
  }
  childOb = observe(newVal)
  dep.notify()
}
```
这里 setter 比较简单，如果有新值就对新值进行 observe 的改造，然后触发通知所有已经被收集的观察者。

## observe
```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value)) {
    return
  }
  // TODO
}
```
`value` 必须是对象才能完成 observe；

```javascript
let ob: Observer | void
if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
  ob = value.__ob__
} else if (
  observerState.shouldConvert &&
  !isServerRendering() &&
  (Array.isArray(value) || isPlainObject(value)) &&
  Object.isExtensible(value) &&
  !value._isVue
) {
  ob = new Observer(value)
}
```
这里我们创建了一个 ob 变量，用于存储 value 的被观察 (Observer) 的实例。如果 value 上已经存在了 __ob__ 属性，则意味着该属性已经被改造为被观察者了。

```javascript
if (asRootData && ob) {
  ob.vmCount++
}
return ob
```
vmCount++ 的含义是如果 value 是作为根组件的数据传递进来，那么让 Vue实例的数量加 1。
最后我们需要返回这个 Observer 的实例。

## Observer
```javascript
constructor (value: any) {
  this.value = value
  this.dep = new Dep()
  this.vmCount = 0
  def(value, '__ob__', this)
  // TODO
}
```
在实例上创建一个 dep 属性，作为观察者收集容器。
这里我们在 value 值上定义 __ob__ 属性，并将生成的实例赋值 __ob__ 上;

```javascript
if (Array.isArray(value)) {
  // 根据浏览器判断 obj.__proto__ 是否存在
  const augment = hasProto
    ? protoAugment
    : copyAugment
  augment(value, arrayMethods, arrayKeys)
  this.observeArray(value)
} else {
  this.walk(value)
}
```
这里比较复杂的是，augment(value, arrayMethods, arrayKeys) 的含义。
首先前提条件是 value 的数据类型应是数组，所以 aument 的作用就是重新定义 value 上部分数组的操作函数。

```javascript
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

/**
 * Intercept mutating methods and emit events
 */
;[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
        inserted = args
        break
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

这段代码里，我只需要关注的核心是 push, pop, shift, unshift, splice, sort, reverse, 这些方法被修改成，一旦数组变动，则会通知相关的观察者们进行更新的动作，并返回正常方法执行的结果。

```javascript
this.observeArray(value);

observeArray (items: Array<any>) {
  for (let i = 0, l = items.length; i < l; i++) {
    observe(items[i])
  }
}
```
实际上就是一次遍历当前 value 每个元素，并对其进行一个 observe 的改造过程。

```javascript
this.walk(value)
```

如果 value 不是数组，那么就是正常的对象，我们同样需要遍历 value 上的每个属性，并对每个属性值进行 defineReactive 的操作，同样将它们全部改造为被观察者。

## 总结

如果数据结构复杂的话，会不断经历 observe => Observer => defineReactive => observe ... 这样类似的过程，直至这个对象的所有属性值都完成被改造，能够在 getter 的时候收集依赖，在 setter 的通知观察者的更新行为。

> [Vue](https://cn.vuejs.org/v2/guide/)
> [Vue源码详细解析(一)--数据的响应化](https://github.com/Ma63d/vue-analysis/issues/1)
