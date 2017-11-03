---
title: 写一个 JavaScript 框架（二）
date: 2017-07-31 14：30
tags: ['翻译', '架构', 'JavaScript']
categories: 翻译
---

这是写一个 JavaScript 框架系列的第二篇。在这个章节中，我将会阐明多种在浏览器中执行异步代码的方式。你们将会了解到事件轮询，各种调度技巧的区别，比如 `setTimeout` 和 `Promises`。

这个系列是关于一个叫 NX 的开源客户端框架。在这个系列中，主要是阐明在开发框架过程中遇见的一些我已克服的难题。如果你对 NX 感兴趣，请阅览[NX 主页](http://nx-framework.com/)。

这个系列包括以下几个章节：
1. [项目结构](http://swarosky44.github.io/2017/07/26/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%80%EF%BC%89/#more)
2. 调度执行（当前章节）
3. [沙箱求值](http://swarosky44.github.io/2017/08/02/%E7%BF%BB%E8%AF%91%20-%20%E5%86%99%E4%B8%80%E4%B8%AA%20JavaScript%20%E6%A1%86%E6%9E%B6%EF%BC%88%E4%B8%89%EF%BC%89/)
4. 数据绑定简介
5. 用 ES6 Proxy 实现数据绑定
6. 自定义元素
7. 客户端路由
<!-- more -->

可能大部分的人都熟悉用 `Promise` , `process.nextTick()`, `setTimeout()` 以及 `requestAnimationFrame()` 执行异步代码。它们都会在内部执行事件的轮询，但是在时间精度上表现出来的差异性十分明显。

这个章节里，我将会解释它们之间的区别，然后向你们展示 NX 是如何实现一个时间系统的。我们不会重新造一个轮子，我们将使用原生的事件轮询去达成我们的目标。

## 事件轮询
事件轮询不仅仅是在 ES6 标准提及，JavaScript 有它自己的任务和任务队列。更复杂的事件轮询在 NodeJS 和 HTML5 标准中有具体说明。由于这是另一个关于前端的系列，我会在后续的文章中提及。

事件轮询之所以叫轮询的原因是，它会不断地循环寻找新的任务去执行。轮询的一次迭代我们称为一个时刻，在这个时刻中执行的代码我们称为一次任务。
```JavaScript
while (eventLoop.waitForTask()) {
  eventLoop.processNextTask();
}
```
在轮询中，一次任务的代码是可以安排其他任务同步执行的。一种简单的手动编程方式去安排一个新的任务是使用 `setTimeout(taskFn)`。尽管如此，任务的来源是多种多样的，比如用户行为，网络请求或者 DOM 操作。

![](http://blog-assets.risingstack.com/2016/Aug/Execution_timing_event_lopp_with_tasks-1470127590983.svg)

## 任务队列
事情更加复杂一点的话，事件轮询可以有多个任务队列。这里有两个限制，同一类的任务必须来自于同一个任务队列，并且任务在每个队列中必须是以插入顺序去执行。除去这些，用户可以自由的做他所想的事。比如，他可以决定下一次执行哪个任务队列。

```JavaScript
while (eventLoop.waitForTask()) {
  const taskQueue = eventLoop.selectTaskQueue();
  if (taskQueue.hasNextTask()) {
    taskQueue.processNextTask();
  }
}
```

在这个模型中，我们放松了对时间精度的控制。在我们的任务使用 `setTimeout()` 之前，浏览器可能已经决定完全清空好几个其他的任务队列了。

![](http://blog-assets.risingstack.com/2016/Aug/Execution_timing_event_loop_with_task_queues-1470127624172.svg)

## 微任务队列
幸运的是，事件轮询还拥有一种单独的队列叫微任务队列。它在每个时刻的任务执行完成之后会完全清空。

```
while (eventLoop.waitForTask()) {
  const taskQueue = eventLoop.selectTaskQueue();
  if (taskQueue.hasNextTask()) {
    taskQueue.processNextTask();
  }

  const microTaskQueue = eventLoop.microTaskQueue;
  while (microTaskQueue.hasNextMicroTask()) {
    microTaskQueue.processNextMicroTask();
  }
}
```

最简单的安排一次微任务的方式是 `Promise.resolve().then(microtaskFn)`。微任务是以插入顺序执行的，由于这里只有一个微队列，这个时候用户是不会混淆我们的。

此外，微任务可以安排新的微任务插入到同一个队列，并在同一个时刻处理。

![](http://blog-assets.risingstack.com/2016/Aug/Execution_timing_event_loop_with_microtask_queue-1470127679393.svg)

## 渲染

最后一件被遗落的事情是渲染计划。不同于事件的处理和分析，渲染并不是由一个单独的后台任务完成的。它是一种可以运行在每一个轮询时刻最后面的算法。
用户因此又拥有许多自由：可以在每个任务之后渲染，也可以让许多任务不用去执行渲染。
幸运的是，我们有 `requestAnimationFrame()`，它在下一次渲染之前执行传递的函数。最终我们的事件模型看上去是这样的：

```JavaScript
while (eventLoop.waitForTask()) {
  const taskQueue = eventLoop.selectTaskQueue();
  if (taskQueue.hasNextTask()) {
    taskQueue.processNextTask();
  }

  const microTaskQueue = eventLoop.microTaskQueue;
  if (microTaskQueue.hasNextTask()) {
    microTaskQueue.processNextMicroTask();
  }

  if (shouldRender()) {
    applyScrollResizeAndCSS();
    runAnimationFrames();
    render();
  }
}
```

![](http://blog-assets.risingstack.com/2016/Aug/Execution_timing_event_loop_with_rendering-1470127703068.svg)

现在让我们用所有的知识去建立一个调度系统。

## 使用事件轮询
和大部分现代框架一样，NX 在后台处理 DOM 操作和数据绑定。NX 分批处理这些操作，并且为了更好的性能，异步执行他们。为了能安排好处理这些操作，NX 依赖于 `Promises`, `MutationObservers` & `requestAnimationFrame()`。

预期的安排有如下：
1. 来自开发者的业务代码；
2. NX 的数据绑定 & DOM 的回应操作；
3. 用户定义的钩子函数；
4. 客户端的渲染；

### 步骤一
NX 同步完成 ES6 Proxies 注册对象 & `MutationObserver` 完成 DOM 变化观察者(这些知识大部分都在下一个章节)。为了优化性能的反馈事件可以通过作为微任务延迟完成。为了观察对象变化的事件可以使用 `Promise.resolve().then(reaction)` 延迟实现，并且通过 MutationObserver (它在内部会调用微任务)自动执行。

### 步骤二
当用户的业务代码执行完成后，通过 NX 注册的响应微任务开始执行。因为它们是微任务，所以它们是按照顺序执行的。需要提醒的是，我们目前仍然是在同一个轮询的时刻里面。

### 步骤三
NX 用 `requestAnimationFrame(hook)` 运行用户传递进来的钩子事件。这个可能发生在下一个轮询的时刻。最重要的是，这些钩子事件运行于下一次渲染之前，所有数据、DOM & CSS 变化已经被处理之后。

### 步骤四
浏览器渲染下一个页面。这个仍然可能发生在下一个轮询时刻，但是它不能发生在上一个步骤的时刻之前。

## 需要牢记于心的事情
我们刚刚基于原生的轮询之上实现了一个简单但是高效的调度系统。理论上它是可以正常工作的，但是调度是非常容易出错的，一些轻微的误差可能造成严重的bug。

在一个复杂的系统中，设立一些调度规则，并且一直坚持是非常重要的。对于 NX，我有以下几个规则：
1. 永远不要在内部操作中使用 `setTimeout(Fn, 0)`;
2. 用同一个方法注册微任务；
3. 微任务只用于存储内部操作；
4. 不要用任何事情去污染开发者的钩子函数执行。

### 规则 1 & 2
数据和 DOM 操作上的响应行为应该按照变化的顺序执行，为了不造成执行顺序混淆，即使延迟执行也是 👌 的。混淆执行顺序使事情变得难以把握并且难以究其原因。`setTimeout(Fn, 0)` 就是完全不可预测的。
通过不同的方式注册微任务也会造成执行顺序混淆，比如下面的案例：微任务2错误的在微任务1之前执行了。

```JavaScript
Promise.resolve().then().then(microtask1)
Promise.resolve().then(microtask2)
```

![](http://blog-assets.risingstack.com/2016/Aug/Execution_timing_microtask_registration_method-1470127727609.svg)

### Rule 3 & 4
区分开发者的任务代码和系统内部操作是非常重要的。将这两个行为混淆在一起可能会导致一些意料之外的行为，而且它还可能会导致开发者不得不去了解框架内部的工作机制。我相信许多前端开发人员已经有过类似的体验了。

## 结论
如果你对 NX 框架感兴趣，可以参阅[主页](http://nx-nxframework.rhcloud.com)。热衷于探索的读者可以在[GitHub Repo](https://github.com/RisingStack/nx-framework)参阅源码。

我希望你能认为这一篇好的文章，下次将会讨论沙箱求值。
