---
title: 数据结构 - Array
date: 2017-03-30 15：05
tags: ['数据结构', 'JavaScript']
categories: 数据结构
---

blog 空空的感觉超级难受 … 多写点多开点坑

数据结构 & 算法作为软件程序的两个基本要素，学好的重要性不言而喻了吧，除非你甘心做一辈子的切图仔 🙃

<!-- more -->

Array 是现代编程语言中最常用的数据结构之一，也是 ECMAScript 已经为我们定义好了的内置的一种数据结构。在原型范式编程语言的万物皆对象的理念下，Array 也是对象。

```javascript
typeof Array // 'object'
```

在很多语言中 Array 在内存空间上是连续性的表现，因此在创建 Array 的时候需要预先分配一片内存空间，Array 的大小（长度）也就无法动态改变，在 Array 内存储的数据如果超出预先分配的内存空间大小，通常会报数据溢出异常。

但是在 ECMAScript 中，Array 的大小是动态的，类似的在 Java 等现代语言中也提供了 ArrayList 等动态数组的数据结构。

## 创建 Array

在 ECMAScript 创建一个 Array 有许多途径：

#### 字面量声明

```javascript
var a = []
var a = [1, 2, 3]
var a = [,,,] // [undefined * 3]
```

通常声明 Array 或者 Object 都推荐使用字面量声明的形式，但是在 JavaScript高级程序设计 中明确指出，并不推荐字面量声明是因为

```JavaScript
var a = [1, 2,] // => 一个可能包含 2 个或者 3个的数组
```

在 Chrome 下这样声明出来的对象 a 是一个包含两个数据的数组，而在 IE8 及之前版本中会被声明成一个包含三个数据的数组。至于要不要推荐字面量声明，就只能看个人想法了 … （反正我是习惯用字面量声明）

#### Array 构造函数

```javascript
var a = new Array() // []
var a = new Array(1, 2, 3) // [1, 2, 3]
var a = new Array(4) // [undefined * 4]
var a = new Array(-1) // Uncaught RangeError: Invalid array length
```

在 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array) 中描述构造函数声明 Array 的语法为

```javascript
new Array(element0, element1, element2...)
new Array(arrayLength)
```

elementN - 数组元素集

arrayLength - 数组长度

这里语法上只有当参数有且仅有一个，并且为 Number 型数值时才会识别为 arrayLength 。

## Array 属性

#### Object 的属性与方法

由于之前提及的，在原型范式下，所有数据都是对象，Array 所构造的对象的原型链终点也是 Object ，所以 Array 也具有了几种通用的方法

```javascript
var a = [1, 2, 3]
a.toString() // '1,2,3'
a.toLocaleString() // '1,2,3'
a.valueOf() // [1, 2, 3]
a.hasOwnProperty('length') // true
```

 #### length 属性

用于表示 Array 长度的属性，但是在真正含义上 length 也无法定义一个 Array 中数据的个数。

因为 length 属性的原属性 - writable 是可读写的，因此我们可以人为的修改一个 Array 的长度 -

当我们将 length 属性值变小时，Array 会截断。

当我们将 length 属性值变大时，Array 只会填充 undefined 到新增索引位置。

```
var a = [1, 2, 3]
a.length = 10
a // [1, 2, 3, undefined * 7]
a[8] = 8
a // [1, 2, 3, undefined * 5, 8, undefined * 1]
```

因此我们给上面这样类似形态的数据结构单独取名为稀疏数组，相反，在一个 Array 中如果每个索引下都是有意义的值，那么我们一般称为稠密数组。

#### Array.prototype

这是每个 array 的构造函数的原型，我们构造的每个数组对象都是克隆的这个对象，值得一提，有趣的是

```javascript
Array.isArray(Array.prototype) // true
```

## Array 方法

### 修改自身的方法

这一类的方法都会修改源数据，所以在使用这些方法然后再赋值的话需要特别注意。

#### Array.prototype.copyWithin

浅复制数组中一部分到同一数组的另一部分, 返回修改之后的源数组。

```javascript
arr.copyWithin(target, start, end)
var arr = [1, 2, 3]
arr.copyWithin(0, 1, 2) // [2, 2, 3]
arr.copyWithin(1) // [1, 1, 2]
```

target - 修改位置的起始索引，如果 `target > arr.length` 则函数不会执行。target 为负数时，从数组末尾开始计数；

start - 复制位置的起始索引，start 为负数时，从数组末尾开始计数；

end - 复制位置的截止索引，end 为负数时，从数组末尾开始计算；

#### Array.prototype.fill

将数组内所有元素从开始索引到结束索引之间填充一个静态值，返回修改后的数组

```javascript
arr.fill(value, start, end)
var numbers = [1, 2, 3]
numbers.fill(1) // [1, 1, 1]
```

value - 用于填充数组的值；

start - 开始索引，为负数时，从数组末尾开始计数；

end - 结束索引，为负数时，从数组末尾开始计算；

#### Array.prototype.pop

删除数组最后一位，返回数组移除的元素，如果数组为空则返回 undefined

```JavaScript
arr.pop()
```

#### Array.prototype.push

在数组末尾添加一位或多位元素，返回新的数组的长度

```javascript
arr.push(1)
arr.push(1, 2, 3)
```

#### Array.prototype.reverse

逆序排序数组中的元素，返回修改后的数组

```javascript
arr.reverse()
```

#### Array.prototype.shift

从数组中删除第一个元素，返回删除的元素

```javascript
arr.shift()
```

#### Array.prototype.sort

在适当的位置对数组进行排序，返回排好序后的数组

```javascript
arr.sort((pre, cur) => {
  // ...
})
```

默认排序是将每个元素转化成字符串，按照 Unicode 码点进行排序。如果期望只是 Number 型数值的排序，会造成一些错误，9 的 Unicode 码点要大于 81 的 Unicode 码点。

所以最好是指定一下排序方法。

pre - cur < 0 则 pre 排在 cur 之前

pre - cur == 0 则 pre 和 cur 的先后关系很难保证确定（不同浏览器结果不一定一致）

pre - cur > 0 则 cur 排在 pre 之前

#### Array.prototype.splice

删除数组中元素，返回删除后元素

```javascript
arr.splice(start, delecount, ...itemN)
var a = [1, 2, 3, 4]
a.splice(0, 2, 'a', 'b') // [1, 2]
a // ['a', 'b', 3, 4]
```

start - 移除元素的起始索引位置，负值从数组末尾开始计数，值超过数组长度函数不执行；

delecount - 从起始位置开始需要删除的元素个数；

…itemN - 在删除元素位置填充的新元素

#### Array.prototype.unshift

将一个或多个新元素添加到数组的开头位置，返回修改后数组的长度

```javascript
arr.unshift('a')
arr.unshift('a', 'b')
```

### 不修改自身的方法

这一类方法不会修改源数组，创建一个新的数组作为返回值

#### Array.prototype.concat

合并拼接两个或多个数组，返回合并后的新数组

```javascript
arr.concat(arr1)
arr.cocnat(arr1, ...arrN)
```

#### Array.prototype.includes

判断当前数组是否包含某指定的值，返回 Boolean 型的结果值

```javascript
arr.includes(value, index)
```

value - 需要查找的元素值

index - 从索引值为 index 的元素位置开始查找

#### Array.prototype.join

将数组中所有元素拼接到一个字符串中，返回这个字符串

```javascript
arr.join(separator)
[1, 2, 3].join('-') // '1-2-3'
```

#### Array.prototype.slice

选取源数组中从起始索引到结束索引之间的元素，浅拷贝到新数组中，返回新数组

```javascript
arr.slice(start, end)
['zero', 'one', 'two', 'three'].slice(1, 3) // ['one', 'two', 'three']
```

start - 起始索引位置，负值从数组末尾开始计数；

end - 结束索引位置，负值从数组末尾开始计数；

#### Array.prototype.indexOf

返回数组中第一个与给定值相等的元素的索引位置，没找到则返回 -1

```javascript
arr.indexOf(value, index)
```

value - 检索的指定值

index - 检索的起始索引值

#### Array.prototype.lastIndexOf

实际原理同 `Array.prototype.indexOf` , 只是从数组右边开始检索

### 遍历方法

遍历数组的方法，通常需要指定一个遍历过程中的执行的回调函数，不一定有返回值

#### Array.prototype.forEach

遍历数组，给每个元素执行一次回调

```javascript
arr.forEach((item, index, arr) => {
  //...
}, thisArg)
```

callback - 回调函数

​	item - 元素对象

​	index - 元素的索引值

​	arr - 元素组

thisArg - 回调函数执行时 this 的指向

#### Array.prototype.entries

将数组的遍历过程变成迭代器，迭代器包含每个元素

```javascript
for (let iter of arr.entries()) {
  console.log(iter) // [index, item]
}

var iterator = arr.entries()
console.log(iterator.next().value)
```

#### Array.prototype.keys

同 `Array.prototype.entries` , 迭代器包含每个元素的索引值

```javascript
var iterator = arr.keys()
iterator.next() // { value: 0, done: false }
```

#### Array.prototype.values

同`Array.prototype.entries` , 迭代器包含每个元素的值

#### Array[Symbol.iterator]

同 `Array.prototype.values` , 同一功能

```javascript
var arr = ['w', 'y', 'k', 'o', 'p'];
var eArr = arr[Symbol.iterator]();
console.log(eArr.next().value); // w
```

#### Array.prototype.every

检测数组内所有元素是否通过了指定的函数测试，只要有一个未通过则返回 `false`

```javascript
arr.every((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.some

检测数组内所有元素中是否有一个元素通过指定的函数测试，只要有一个通过则返回 `true`

```javascript
arr.some((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.filter

返回数组内所有通过指定函数测试的元素（返回值为 `true` ）所组成的数组

```javascript
arr.filter((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.find

返回数组中满足指定的函数测试的第一个元素，否则返回 `undefiend`

```javascript
arr.find((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.findIndex

同 `Array.prototype.find` , 返回元素的索引值，未找到则返回 -1

```javascript
arr.findIndex((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.map

创建一个新的数组，每个元素都是源数组中的元素经过指定的函数处理后的结果

```javascript
arr.map((item, index, arr) => {
  //...
}, thisArg)
```

#### Array.prototype.reduce

对数组每个元素按照指定的函数进行累加操作，返回为单个值

```javascript
arr.reduce((item, index, arr) => {
  //...
}, initialValue)
```

initiaValue - 累加值的初始值

#### Array.prototype.reduceRight

同 `Array.prototype.reduce` , 按照从右至左的顺序累加