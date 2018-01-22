---
title: Map & Set
date: 2018-01-11 10：00
tags:
  - ES6
  - JavaScript
categories: ES6
---

很久没有写过关于 ES6 和 JavaScript 的学习笔记了。Java 中的集合框架中对于 Set & Map 提供了十分成熟且丰富的类库，如：HashSet、TreeSet、HashMap、TreeMap 等...
在 JavaScript 开发过程中，我可能还是习惯上用 Array + Object 当做万能的数据结构，但是对于 Set 和 Map 的优越性却了解不多。

<!-- more -->

## Set
对于 Set 的定义：一种包含不同元素的数据结构，允许你存储任何类型的唯一值，无论是原始值或者是对象引用。通常我们称 Set 为集合。

### Set 的构造函数
Set 的构造函数就是自身：`var set = new Set();` 这是 `set` 的值为 `Set(0) {}`，构造实例的时候需要传递一个可迭代对象为参数。

### Set 的特性
1. Set 最大的特征就是集合内没有重复值，不论是构造实例时候还是之后同 `add` API 添加新元素的时候，如果集合中已有重复值就会忽略。这里判断是否为重复值的实现类似于 "==="，唯一不同的是会认为 NaN === NaN。对于 Set 集合内没有重复值，我们可以用于实现去重，这样实现的简单程度要远胜于之前用 Array 来存储数据和实现去重算法：
```JavaScript
let arr = [...new Set(array)];
```


2. Set 是一个有序的可迭代对象，我们可以按照添加元素的顺序去迭代它的元素，我们可以通过 `for ... of` 来遍历一个 set 实例，Set 的原型也为我们提供了 forEach、entries、values、keys 等遍历实例的方法。

### Set 的 API
- `Set.prototype.size`：返回 Set 集合中元素的个数；

- `add(value)`：添加某个值，这个值可以为任意数据类型，返回 Set 实例本身。

- `delete(value)`：删除某个值，返回一个 boolean，表示操作是否成功。

- `clear()`：清空 set 集合中所有元素，无返回值。

- `keys()`：返回键名的迭代器对象。eg：
```JavaScript
  let set = new Set([1, 2, 3]);
  for (let key of set.keys()) {
    console.log(key);
  }
  // 1
  // 2
  // 3
```

- `values()`：行为和 `keys()` 完全一致，因为 Set 是值值对应的数据结构，也就是说键名和键值是同一个值。

- `entries()`：返回同时包含键名和键值的迭代器对象。eg：
```JavaScript
let set = new Set(['blue', 'orange', 'green']);
for (let item of set.entries()) {
  console.log(item);
}
// ['blue', 'blue']
// ['orange', 'orange']
// ['green', 'green']
```

- `forEach(callback)`：遍历集合，对每个元素都执行 callback 函数操作，无返回值;

## Map
对于 Map 的定义：用于保存键值对的数据结构，任何值都可以作为一个键或一个值。通常我们称之为字典。

### Map 的构造函数
Map 的 构造函数就是自身 `var map = new Map()`; 接受一个可迭代对象为参数，但是需要满足键值对的形式，eg:
```JavaScript
// 传递数组
let map = new Map([['a', 1], ['b', 2]])
// map: [{key: 'a', value: 1}, { key: 'b', value: 2 }]

// 传递 Set
let map = new Map(new Set([1, 2]))
// map: [{ key: 1, value: 1 }, { key: 2, value: 2 }]
```

### Map 与 Object 的异同
从很多角度上来看，Map 对象和 Object 十分相似，都是存储键值对的数据结构，键值都是唯一，重复的键值会发生后者覆盖前者。都可以删除键值对，也可以查询是否存在指定键值。所以不得不承认，很多时候因为 ES5 以前的 ES 的缺失性，很多前端开发人员都在把 Object 当作 Map 使用。所以这里我们简单区分一下两者的差异，并且思考一下何时使用 Map 是更优的方案：

1. Map 的键名不局限于 String & Symbol 两个数据类型，Map 的键名可以是任意原始数据类型，也可以是任意对象。在键名的唯一性上和 Set 是一致的，类似于 "==="，但是认为 NaN === NaN。

2. 是一个有序的可迭代对象，可以通过 `for ... of` 的形式完成 Map 实例的遍历。eg:
```Javascript
let map = new Map([['aa', 'xxx'], ['bb']]);
for (let item of map) {
  console.log(item);
}
// ['aa', 'xxx']
// ['bb', undefined]
```
3. Map 提供 API 可以十分方便的获取字典内所键值对的个数，Object 则比较麻烦，`var keys = Object.keys(obj).length;`。

因此我们可以看到很多方面 Map 相对于 Object 有很明显的优势，不过不代表所有场景我们都该用 Map, 使用 Map 前最好确认下需求：
- 是否需要动态地查询 key ?
- 所有的值是否都是统一类型 ？
- 是否有些 key 不能是字符串类型或者 Symbol 类型 ？
- 键值对是否经常要发生删改 ？
- 集合是否需要遍历功能 ？

如果有多个需求为 YES，那么我应该优先使用 Map。

### Map 的 API
- `Map.prototype.size`：返回 Map 对象的键值对的数量。

- `clear()`：移除 Map 对象上所有键值对。

- `delete(key)`：移除指定键名，返回键值。

- `get(key)`：返回键名对应的键值，不存在键名则返回 undefined。

- `has(key)`：返回布尔值，表示 Map 实例中是否存在该键名。

- `set(key, value)`：向 Map 实例设置新的键值对，返回该 Map 实例。

- `entries()`：返回一个可迭代对象，按插入顺序包含了 Map 实例中每个元素的 [key, value] 数组。

- `values()`：返回一个可迭代对象，按插入顺序包含了 Map 实例中每个元素的键值。

- `keys()`：返回一个可迭代对象，按插入顺序包含了 Map 实例中每个元素的键名。

- `forEach(callback)`：返回一个可迭代对象，按插入顺序对 Map 实例中每个元素进行 callback 操作。

## 总结
以上就是 Set & Map 的学习记录，可以发现 Set & Map 相对于以往所习惯的 Array 和 Object 有许多优势，编码前考虑好场景问题，使用正确的数据结构，对于构建可维护性强壮的代码的意义十分重要。