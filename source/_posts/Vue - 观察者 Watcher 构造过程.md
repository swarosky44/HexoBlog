---
title: Vue - 观察者 Watcher 构造过程
date: 2017-07-01 18：00
tags:
  - Vue
  - JavaScript
categories: Vue
---

写过上一次的 [Vue 响应式数据构造过程](http://swarosky44.github.io/2017/06/30/Vue%20-%20%E5%93%8D%E5%BA%94%E5%BC%8F%E6%95%B0%E6%8D%AE%E6%9E%84%E9%80%A0%E8%BF%87%E7%A8%8B/#more) 的文章后，感觉确实能帮助理解不少。所以再接再厉，撸完之前避开没有谈及的 Watcher（观察者）构造的过程。

<!-- more -->

## 预备知识
同样先给出一些预备的知识点，尤其是响应式数据构造的知识，因为 watcher 的实现中会有发生交集的地方。

- 观察者模式：[图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/behavioral_patterns/observer.html) & [我的文章](http://swarosky44.github.io/2017/05/02/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/#more)
- 发布者构造：[Vue 响应式数据构造过程](http://swarosky44.github.io/2017/06/30/Vue%20-%20%E5%93%8D%E5%BA%94%E5%BC%8F%E6%95%B0%E6%8D%AE%E6%9E%84%E9%80%A0%E8%BF%87%E7%A8%8B/#more)

## $watch

我是从 Vue.prototype.$watch 这个 API 切入的。实际上 .vue 文件中的 watch 选项也是通过遍历对象属性，然后依次调用 $watch 实现的。

```javascript
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  // TODO
}
```

我们先看一下这里三个参数：
expOrFn： 我们观察的对象。如果是字符串，代表某个 obj 上的属性的键路径；如果是函数，代表计算属性，(tip: 运行时已经绑定了 vm 为 this 的志向，因此不可以是箭头函数)。
cb：回调函数，用于在 watcher 接收到更新信息后执行的函数，就是 watch 内定义的函数。
options：观察者创建时的配置项，有 deep、immediate 等选项。

```javascript
const vm: Component = this
if (isPlainObject(cb)) {
  return createWatcher(vm, expOrFn, cb, options)
}
options = options || {}
options.user = true
const watcher = new Watcher(vm, expOrFn, cb, options)
if (options.immediate) {
  cb.call(vm, watcher.value)
}
return function unwatchFn () {
  watcher.teardown()
}
```

再看下内部的实现：
- 我们先判断参数 cb 是不是对象，如果是对象的话则调用 createWatcher 函数，这里这个函数非常简单，只是用于寻找一个真正为函数的 cb，然后重新调用 vm.$watch 方法，确保 cb 被执行时一定是函数。这里如果 cb 是对象则会寻找 handler 属性的值。如果 cb 是字符串的话，则会选择 vm 实例上已经注册了的方法（列如：method 内注册的方法）。

- `options.user` 属性是用于区分生产环境和开发环境下一些警告语句的输出。

- 创建了一个观察者的实例。

- 如果配置项 `options.immediate` 为 true，则立即以观察的对象的初始值来执行回调函数。

- 最后返回的一个工具函数，在官网 API 里面有提及，是用于取消观察者的工具，可以让一个 watcher 实例失效。

## Class Watcher

这里我们将类里面的每个方法都逐个看一下。

### constructor

```javascript
this.vm = vm
vm._watchers.push(this)
```
在当前 Vue 的实例 `vm._wathcers` 上收集这个观察者。

```javascript
if (options) {
  this.deep = !!options.deep
  this.user = !!options.user
  this.lazy = !!options.lazy
  this.sync = !!options.sync
} else {
  this.deep = this.user = this.lazy = this.sync = false
}
this.cb = cb
this.id = ++uid // uid for batching
this.active = true
this.dirty = this.lazy // for lazy watchers
this.deps = []
this.newDeps = []
this.depIds = new Set()
this.newDepIds = new Set()
this.expression = process.env.NODE_ENV !== 'production'
  ? expOrFn.toString()
  : ''
```
初始化 options 上的各项参数（用户未配置则初始化为 false）：
  - deep 决定 `wathcer.value` 上的值是否深度观察。
  - user 区分某些情况下是否应给出警示或者错误提示。
  - lazy 是否延迟或者 expOrFn 的值，如果延迟的话，必须等待外部调用 `watcher.get` 后才会获取 expOrFn 的值，然后赋值到 `watcher.value` 上。
  - sync 决定是立即执行回调函数，还是进入更新队列，等待执行。

初始化 watcher 的 ID（ID：step 为 1 递增）；
初始化 deps、newDeps、depIds、newDepIds，这些是用于记录该观察者都被哪些观察者容器收集了（或者说当前的观察者都观察了哪些 Observer），其中 deps 和 depIds 是真正用于收集的容器，而 newDeps 和 newDepIds 则类似于一个缓冲容器，当 cleanupDeps 执行时完成清空缓冲，赋值容器的工作；

```javascript
if (typeof expOrFn === 'function') {
  this.getter = expOrFn
} else {
  this.getter = parsePath(expOrFn)
  if (!this.getter) {
    this.getter = function () {}
    process.env.NODE_ENV !== 'production' && warn(
      `Failed watching path: "${expOrFn}" ` +
      'Watcher only accepts simple dot-delimited paths. ' +
      'For full control, use a function instead.',
      vm
    )
  }
}
```
之前有说过，如果 expOrFn 是键路径的字符串：

我们可以简单看一下 parsePath 的实现（util/lang.js）

```javascript
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```
这里很简单，我们先解析路径，随后利用闭包将 segments 推出parsePath 的生命周期，使其存在内存中，当闭包调用时，就可以调用这个路径数组，返回我们想要的值。

如果 expOrFn 本身就是计算属性的函数的话，就可以在调用时，直接返回一个值给我们。
如果 expOrFn 两者都不符合，则会给出警示。

```javascript
this.value = this.lazy
  ? undefined
  : this.get()
```

这里比较简单，就是判断是否需要延迟计算，如果不需要的话，就直接计算 expOrFn 的值，然后保存在 value 属性上。

### get
```javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  if (this.user) {
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    }
  } else {
    value = this.getter.call(vm, vm)
  }
  // "touch" every property so they are all tracked as
  // dependencies for deep watching
  if (this.deep) {
    traverse(value)
  }
  popTarget()
  this.cleanupDeps()
  return value
}
```
这里用于获取 expOrFn 的值。
- 首先我们将当前的观察者推送到 Dep.target 内，为了让 Vue 能够完成内部依赖收集的工作。（详见 defineReactive 的 getter 的实现）。
- 计算 getter，获取值。
- 如果 options.deep 为 true，则开启深度依赖收集。随后会深入介绍，这里和 observe 之间的互动比较隐晦。
- 从 Dep.target 推出当前的观察者。
- 同步容器。
- 返回 getter 计算值。

#### traverse
```javascript
const seenObjects = new Set()
function traverse (val: any) {
  seenObjects.clear()
  _traverse(val, seenObjects)
}

function _traverse (val: any, seen: ISet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || !Object.isExtensible(val)) {
    return
  }
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```
这里简单来看，就是一次单纯的递归。但是 `while (i--) _traverse(val[i], seen)` & `while (i--) _traverse(val[keys[i]], seen)` 是会触发我们之前在 defineReactive 中定义的 getter, 如下：

```javascript
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
```
因此 val 下的每个紫属性都会完成依赖收集，只要是 val 下的子属性发生变化时都会触发 val 的 update 以及回到函数的执行。
因此说这里有个非常隐晦的互动关系。

### addDep
```javascript
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      // this => watcher || Dep.target
      dep.addSub(this)
    }
  }
}
```
这里需要提前说明白的是：Dep 类 与 Watcher 类 的关系是 N 对 N 的关系。也就是两者之间的联系是如果你收集了我，我也必须收集你。
所以这里当要给 watcher 实例添加一个新的观察，则先收集它的观察者容器。然后让自己被这个观察者容器收集。

### cleanupDeps
```javascript
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  let tmp = this.depIds
  this.depIds = this.newDepIds
  this.newDepIds = tmp
  this.newDepIds.clear()
  tmp = this.deps
  this.deps = this.newDeps
  this.newDeps = tmp
  this.newDeps.length = 0
}
```
这里的工作其实就是同步一下 缓冲容器 和 真正容器。在真正容器中移除缓冲容器中不存在的对象，将缓冲容器中的值赋予真正容器，并清空缓冲容器。

### update
```javascript
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```
这里就是接受 Observe 实例的更新通知的函数了，根据 sync 的值我们判断是立即更新还是进入更新队列，等待更新。

###
```javascript
run () {
  if (this.active) {
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      if (this.user) {
        try {
          this.cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        this.cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

这里就是执行 watcher 的回调，同时传入新旧值。

### evaluate
```javascript
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

暂时没遇见过这里的调用环境，不太明确，只能理解为获取 getter 的值。

### depend
```javascript
depend () {
  let i = this.deps.length
  while (i--) {
    this.deps[i].depend()
  }
}
```
这其实就是一次从 watcher 开始触发的 Dep 和 Watcher 之间依赖互相收集的过程。

### teardown
```javascript
teardown () {
  if (this.active) {
    // remove self from vm's watcher list
    // this is a somewhat expensive operation so we skip it
    // if the vm is being destroyed.
    if (!this.vm._isBeingDestroyed) {
      remove(this.vm._watchers, this)
    }
    let i = this.deps.length
    while (i--) {
      this.deps[i].removeSub(this)
    }
    this.active = false
  }
}
```
这里就是一个 watcher 实例的取消工具，在 `vm._watchers` 上移除自己，同时通知 deps 中所有 dep 移除自己，然后清空 deps。

## 总结
以上就是一个 Watcher 的所有构造过程，以及工具函数的使用。自己仔细去读的话可以理解很多，一方面是如果架构这样一种非常大型的观察者模式，同时看看相关工具函数，学学高手的高级招数也是很不错的。

这里 Watcher 主要表现的业务场景还是 watch 这个选项。但是随着之后源码更多深入挖掘，应该会发觉更多的使用场景。😎
