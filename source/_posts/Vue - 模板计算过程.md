---
title: Vue - 模板计算过程
date: 2017-07-14 14：30
tags:
  - Vue
  - JavaScript
categories: Vue
---

Vue 的模板计算过程应该是 Vue 源码中最大头的东西，也是响应式数据外另一个核心。
其实自己也没读完，或者弄得非常明白，所以打算边写边读，加深理解。

<!-- more -->

这篇主要写 Vue 在模板计算前的一些准备工作。以及其触发的场景之类的。

## 渲染前的调用过程

### new Vue()

当我们启动一个 Vue 项目的时候，毫无意外 `new Vue(options)` 会是一切的起点，而 new 一个 Vue 的实例实际上主要执行的是 `Vue.prototype._init`。

### init

`Vue.prootytpe._init` 在 core/instance/init.js 中定义。

`_init` 函数做的事情很多，初始化各个参数，调用各种底层 API， 调用各个生命周期函数，最后会执行 `vm.$mount(vm.$options.el)`。从这里开始挂载我们的模板到页面上，但在此之前还需要解析模板上各种自定义的指令。

### $mount

`Vue.prototype.$mount` 在 platforms/web/runtime/index.js 文件中定义。


```JavaScript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```
这里简单说一下 hydrating 这参数，我们是不需要太关心的，因为这是 vue-ssr 渲染时才会用上，所以之后我们都会忽略这个参数。
这里我们发现 $mount 只是一层装饰，核心在于 `mountComponent` 这个函数的执行。

在 platforms/web/entry-runtime-with-compiler.js 中会进行一层装饰
```JavaScript
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // Compile TODO
  return mount.call(this, el, hydrating)
}
```
这里省略的代码就是整个模板从计算 -> 渲染 -> 挂载中的第一步：计算。
这里将一个 render Function 的字符串形式准备好，存储到 vm._render 中，等待模板的下一步处理过程。

### mountComponent

`mountComponent` 定义在 /core/instance/lifecycle.js。

`mountComponent` 这个函数主要做以下几件事：
  - 保存挂载的父元素到 $el 属性上；
  - 如果我们传递的 options 里面没有定义 render 函数，则默认为生成一个空的 vnode 实例。并且非生产环境会报错；
  - 执行 `beofreMount` 生命周期的钩子函数；
  - 定义 updateComponent, 作为 expOrFn 保存到一个 Watcher 的实例中，实例保存到 `vm._watcher` 的属性中;
  - 执行 `mount` 生命周期的钩子函数；

这里最核心的就是下面的代码：
```JavaScript
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  updateComponent = () => {
    const name = vm._name
    const id = vm._uid
    const startTag = `vue-perf-start:${id}`
    const endTag = `vue-perf-end:${id}`

    mark(startTag)
    const vnode = vm._render()
    mark(endTag)
    measure(`${name} render`, startTag, endTag)

    mark(startTag)
    vm._update(vnode, hydrating)
    mark(endTag)
    measure(`${name} patch`, startTag, endTag)
  }
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}

vm._watcher = new Watcher(vm, updateComponent, noop)
```

这段函数最核心的地方就是定义好了 updateComponent，并生成一个 Watcher 的实例，这里定义一个 watcher，那么它的 value 就是我们最终模板生成的 dom，而 expOrFn 就是我们更新 dom 的函数，看到最后面就会明白这是一个怎样的闭环了。

在 `/core/instance/lifecycle.js` 中的 `lifecycleMixin` 中我们就可以看到 `Vue.prototype.$forceUpdate` 是通过调用 watcher 实例上的 update 方法完成的更新视图。

而 updateComponent 函数的定义也有几个地方需要了解一下：
  - vnode 参数的准备来自于 `Vue.prototype._render`，用于将模板的 DOM 转换成 vnode 对象。
  - `vm._update` 就是继续向下承接的函数，会在那里面完成最终的渲染（这一层层的函数包裹，哇，看得我头皮发麻 🙄 ）。

### update

`Vue.prototype._update` 定义在 /core/instance/lifecycle.js。

```JavaScript
if (!prevVnode) {
  // initial render
  vm.$el = vm.__patch__(
    vm.$el, vnode, hydrating, false /* removeOnly */,
    vm.$options._parentElm,
    vm.$options._refElm
  )
  // no need for the ref nodes after initial patch
  // this prevents keeping a detached DOM tree in memory (#5851)
  vm.$options._parentElm = vm.$options._refElm = null
} else {
  // updates
  vm.$el = vm.__patch__(prevVnode, vnode)
}
```

这里的 __patch__ 就是最后我们所说的那个大家伙了，之前所有的工作，都是只是准备工作。
这里很简单的一个判断是，当前渲染的 vm 实例是根组件，还是某个子组件。

### patch

`Vue.prototype.__patch__` 定义比较繁复，从 platform/runtime/index.js 开始，到 ./patch.js（这里有两个参数 nodeOps & modules 稍后会讲一下）最后在 core/vdom/patch.js 中看到完完整整的实现。

```JavaScript
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

```JavaScript
return function patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {
  // TODO
  return vnode.elm
}
```

最后我们返回了一个闭包函数，所以 `vm.__patch__` 实际上是 createPatchFunction 的返回结果，而 `vm.__patch__` 的执行结果是一个真实的 DOM 树。

所以最后我们可以看到 patch 执行前的主要工作是函数间的不断调用，每个函数单元又会做一些些准备。

## 两个工具

```JavaScript
export const patch: Function = createPatchFunction({ nodeOps, modules })
```

就是这里的两个参数，nodeOps 和 modules。

- nodeOps 是一组 DOM 操作方法的集合；
- modules 是一组指令生命周期的方法的集合；

### nodeOps
在 /platforms/web/runtime/node-ops.js 文件中，我们可以看到大量 DOM 操作方法，这里就是一个工具类，在 patch 函数计算模板的过程中，会经常用到这些工具。

### modules
modules 是一个合并数组，由 core/vdom/modules/index.js & web/runtime/modules/index 中的多个指令解析工具组合而成。
最终是这样一个数组
```JavaScript
modules = [
  attrs,
  klass,
  events,
  domProps,
  style,
  transition,
  ref,
  directives,
]
```

而每个对象，都是类似的结构：

```JavaScript
{
  create: updateAttrs,
  update: updateAttrs,
}
```

这里是有约定在的，约定每个解析指令的工具都提供相同命名的钩子，在 patch 执行过程中用于解析模板上的各种指令。

## 总结
其实整个模板计算的闭环都已经讲完了，但是依然晕的不行，各种文件翻来翻去，各种调用，一大堆不知道什么时候就定义好的参数和方法。哇，真的是翻皮水 ...

所以做了个脑图，抽离掉一些不那么重要的函数包裹，看一下代码核心逻辑是怎么流动的。

![](http://7xro5v.com1.z0.glb.clouddn.com/vue%20patch%20%E6%A8%A1%E6%9D%BF%E8%AE%A1%E7%AE%97%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
就先这样吧，之后再写 `function patch` 里面的逻辑吧。😶
