---
title: 数据结构 - Stack
date: 2017-03-30 15：05
tags: ['数据结构', 'JavaScript']
categories: 数据结构
---

Stack 的中文名字是 栈。作为一种数据结构，并不像数组一样在高级语言中有内置的实现，今天就动手定义一个 Stack 的实现类 🤔 。

<!-- more -->

## 定义

Stack 是一种遵守先进后出（LIFO，Last in First Out）规则的有序的集合。在大多数的书籍中对它的实现都采用数组作为这个有序集合的实现。因此我们可以理解为一种有特殊规则的数组（实际上我们也可以用链表来实现一个 Stack）。

既然采用数组来实现一个栈，那么必然在内存中也是以一片连续的内存空间进行存储的。如果在连续的内存空间十分紧张的时候，可以考虑采用链表实现一个 Stack ，这样在内存的存储形式上就是离散的，碎片化的。

## 实现

```javascript
class Stack {
  constructor () {
    this.items = []
  }
  
  push (element) {
    this.items.push(element)
  }
  
  pop () {
    return this.items.pop(element)
  }
  
  peek () {
    return this.items[this.items.length - 1]
  }
  
  isEmpty () {
    return this.items.length === 0
  }
  
  clear () {
    this.items = []
  }
  
  size () {
    return this.items.length
  }
}
```

十分简单的一种数据结构，对于类内部的方法，我们发现大多数都是直接使用数组的内置函数。但是不要奇怪，这样子封装在多人项目中能起到非常好的风险把控效果，在业务场景中，能有效阻止一些新人或者脑子忘在家里的同事做一些某些数据不该做的事。

接下来我们试着使用一下这个类，实现一个十进制转二进制的过程。

```javascript
const divideBy2 = decNum => {
  let stack = new Stack()
  let num = parseInt(decNum)
  let rem = 0
  let result = ''
  
  while (num > 0) {
    rem = num % 2
    stack.push(rem)
    num = Math.floor(num / 2)
  }
  while (stack.size() > 0) {
    result += stack.pop()
  }
  return result
}

divideBy2(11) // "1101"
divideBy2(1111.11) // "10001010111"
```

对于栈的实现与使用就是这么简单啦 ~ 这么多写下来，只有一点需要努力去记忆，就是 LIFO 是先进后出！！！

## BTW

值得一提的是，栈不仅仅是一种数据结构中的术语，在计算机原理中，也是一种内存存储数据的内存区域的术语名词，如果感兴趣可以看看 [阮老师的 Stack 的三种含义](http://www.ruanyifeng.com/blog/2013/11/stack.html) 。

