---
title: Vue - 实例事件构造过程
date: 2017-07-13 18：00
tags:
  - Vue
  - JavaScript
categories: Vue
---

有段时间没写关于 Vue 源码的东西了，最近都在折腾 python 的 flask & vue-ssr。

<!-- more -->

## 基础知识
相比之前 vue 的观察者模式的构建过程，这里的实例事件就要显得简单多了。不需要了解什么特别的设计模式啊或者其他balbala的知识点。熟悉一下官方的 API 就好了。

- [vue 实例方法/事件](https://cn.vuejs.org/v2/api/#实例方法-事件)

## 实现
整个事件的体系构造是在 eventsMixin 里面完成的。

```javascript
export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    // TODO
  }

  Vue.prototype.$once = function (event: string, fn: Function): Component {
    // TODO
  }

  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    // TODO
  }

  Vue.prototype.$emit = function (event: string): Component {
    // TODO
  }
}
```

根据 VUE 官网的 API ，很容易就知道这四个函数分别定义用来处理什么问题的，接下来就一个个来分析咯。

### $on
绑定事件和执行事件的方法；

```JavaScript
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      this.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
    // optimize hook:event cost by using a boolean flag marked at registration
    // instead of a hash lookup
    if (hookRE.test(event)) {
      vm._hasHookEvent = true
    }
  }
  return vm
}
```
先简单看一下参数：
- event：需要绑定的事件名，可以为字符串或者数组；
- fn: 需要绑定的执行函数；

如果 event 是数组的话我们需要遍历每个元素，然后重新调用 $on 分别对每个元素进行绑定工作，我们在 Vue 的实例 vm 上创建了一个属性 `_events` 用于存储这些事件以及其对应的执行方法。这里需要注意的是：

一个函数可能会有多个事件都会去绑定，同样一个事件也可能绑定多个函数。

### $off
取消事件和执行函数之间的互相绑定关系。

```javascript
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
  const vm: Component = this
  // all
  if (!arguments.length) {
    vm._events = Object.create(null)
    return vm
  }
  // array of events
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      this.$off(event[i], fn)
    }
    return vm
  }
  // specific event
  const cbs = vm._events[event]
  if (!cbs) {
    return vm
  }
  if (arguments.length === 1) {
    vm._events[event] = null
    return vm
  }
  // specific handler
  let cb
  let i = cbs.length
  while (i--) {
    cb = cbs[i]
    if (cb === fn || cb.fn === fn) {
      cbs.splice(i, 1)
      break
    }
  }
  return vm
}
```

这里取消绑定和绑定的参数没有什么区别，我们可以直接看执行过程：

- 这里首先判断了是否有参数 `if (!arguments.length)`，这里其实是尤大自己用来调用的，如果没有传递参数就直接清空 `vm._events`，也就是说直接取消掉了所有的事件绑定。尤大自己在组件 `$destroyed` 调用时会这样用。

- 正常情况下，使用者如果传递的 event 是数组，就意味着有多组事件绑定需要取消，所以要遍历对每个元素执行 $off，分别取消。

- 如果只传了事件名，但是不指定取消哪个执行函数的话， 就会直接清空这个事件名下所有的绑定的执行函数。

- 在指定了函数名 & 执行函数的情况下，会找到 `vm._events` 里这个 `event`，然后从它里面再找到指定的函数，移除掉。

### $once
绑定事件，但只允许执行一次

```javascript
Vue.prototype.$once = function (event: string, fn: Function): Component {
  const vm: Component = this
  function on () {
    vm.$off(event, on)
    fn.apply(vm, arguments)
  }
  on.fn = fn
  vm.$on(event, on)
  return vm
}
```

装饰了一下调用者传递进来的执行函数，绑定成功后，第一次执行的时候，装饰函数就会取消掉绑定。

### $emit
触发事件

```javascript
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  if (process.env.NODE_ENV !== 'production') {
    const lowerCaseEvent = event.toLowerCase()
    if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
      tip(
        `Event "${lowerCaseEvent}" is emitted in component ` +
        `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
        `Note that HTML attributes are case-insensitive and you cannot use ` +
        `v-on to listen to camelCase events when using in-DOM templates. ` +
        `You should probably use "${hyphenate(event)}" instead of "${event}".`
      )
    }
  }
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    for (let i = 0, l = cbs.length; i < l; i++) {
      try {
        cbs[i].apply(vm, args)
      } catch (e) {
        handleError(e, vm, `event handler for "${event}"`)
      }
    }
  }
  return vm
}
```

- 首先判断用户是否已经完成了指定事件的绑定。
- 然后从 `vm._events` 这个事件存储容器中找到对应事件，再遍历执行该事件下已经绑定了的执行函数们。

## 总结
实例事件的构造还是很简单的，但是接下来就会有两个复杂度非常高的模块，模板计算和渲染。
我读了两次，放弃了两次。哈哈哈 🙄
