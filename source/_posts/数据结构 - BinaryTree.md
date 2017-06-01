---
title: 数据结构 - Binnary Tree
date: 2017-06-01 15：05
tags: ['数据结构', 'JavaScript']
categories: 数据结构
---

话不多说，动手撸🌲 ...

<!-- more -->

## 定义
二叉树（Binnary Tree）是每个节点不能拥有超过两个子节点的树结构。

树结构的定义可以在 [维基百科](https://zh.wikipedia.org/wiki/%E6%A8%B9%E7%8B%80%E7%B5%90%E6%A7%8B) 了解一下。

## 关键词
根节点：树最上面的节点；
父节点：如果一个节点下有其他分支节点，那么该节点就称为父节点；
子节点：父节点下面直接连接的节点；
叶子节点：如果一个节点下面没有其他分支节点，那么该节点就称为叶子节点；
边：父节点与子节点链接的线；
深度：树的层数；

## 实现
接下来我们以 `[50, 40, 60, 30, 45, 47, 5, 21, 2, 94, 83, 25, 32, 43, 99, 100, 34]` 这组值，构建一个二叉树。

首先我们要需要先构建一个类用来当作树的节点。

```javascript
class Node {
  constructor(data) {
    this.data = data;
    this.left = null;
    this.right = null;
  }
}
```

先构建一个简易的二叉树，能够实现插入、删除、遍历即可；

```javascript
class BTS {
  constructor() {
    this.root = null;
  }
}
```

### 实现插入功能。
```javascript
insert(data) {
  const node = new Node(data);
  if (this.root === null) {
    this.root = node;
  } else {
    let current = this.root;
    let parent;
    while (true) {
      parent = current;
      if (data < current.data) {
        current = current.left;
        if (current === null) {
          parent.left = node;
          break;
        }
      }
      if (data > current.data) {
        current = current.right;
        if (current === null) {
          parent.right = node;
          break;
        }
      }
      if (data === current.data) {
        throw new Error('树中已存在该值, 不允许重复存储！');
        break;
      }
    }
  }
}
```
以上代码逻辑是一个简单的迭代过程：
1. 根据传入的值创建一个节点对象(node)。
2. 判断根节点是否为空，如果为空，则当前传入的节点作为根节点存储。反之将根节点存储在变量 current 中，标记为本次迭代中的需要判断的当前节点。
3. 如果 node < current 为真，则判断 current.left 是否存在，不存在，则保存 node 节点。反之进入下一次迭代。node > current 同理。
4. 这样循环直到 node 被保存在某个子树的左（或右）节点之中截止。
5. 如果当前树中已经存在值相同的节点，则冒泡错误，终止迭代过程。

### 实现删除功能
```javascript
remove(data) {
  this.root = this.removeNode(this.root, data);
}

removeNode(node, data) {
  if (node === null) {
    return null;
  }
  if (Data === node.data) {
    // 该节点没有节点
    if (!node.left && !node.right) {
      return null;
    }
    // 该节点有左节点，没有右节点
    if (node.left && !node.right) {
      return node.left;
    }
    // 该节点有右节点，没有左节点
    if (!node.left && node.right) {
      return node.right;
    }
    // 该节点存在两个节点
    const tempNode = this.getMin(node.right);
    node.data = tempNode.data;
    node.right = this.removeNode(node.right, tempNode.data);
    return node;
  } else if (data < node.data) {
    node.left = this.removeNode(node.left, data);
    return node;
  } else {
    node.right = this.removeNode(node.right, data);
    return node;
  }
}
```
删除节点是比较麻烦的，在于需要递归（或迭代）找到对应节点，然后再根据找到节点的子节点数目做相应处理：
1. 如果存在左节点、而不存在右节点，则将左节点作为当前节点返回；
2. 如果存在右节点、而不存在左节点，则将右节点作为当前节点返回；
3. 如果左右节点都存在，则寻找左子树中最大值（或右子树中最小值）作为当前节点返回，这是因为这两个值都是最接近当前节点值得存在；
4. 如果左右节点都不存在，则将 null 作为当前节点返回；

其实删除节点并不是简单的删除某个节点，而是将整个树重新拼接的过程，每次递归都是不断在划分子树，然后返回这个子树，最终树结构重新拼接完成。

### 实现遍历
```javascript
// 中序遍历 - 先访问左子树，再访问根节点，最后访问右子树；
inOrder(node, fn) {
  if (node !== null) {
    this.inOrder(node.left, fn);
    fn.call(this, node);
    this.inOrder(node.right, fn);
  }
}

// 先序遍历 - 先访问根节点，再访问左子树，最后访问右子树；
preOrder(node, fn) {
  if (node !== null) {
    fn.call(this, node);
    this.preOrder(node.left, fn);
    this.preOrder(node.right, fn);
  }
}

// 后序遍历 - 先访问左子树，再访问右子树，最后访问根节点；
afterOrder(node, fn) {
  if (node !== null) {
    this.afterOrder(node.left, fn);
    this.afterOrder(node.right, fn);
    fn.call(this, node);
  }
}
```
二叉树的遍历真是...其实三种遍历都不复杂，画一个简单的二叉树，然后跟着代码逻辑梳理一下就很清晰了。

### 实现查找功能
```javascript
find (data) {
  let current = this.root;
  while (current !== null) {
    if (current.data === data) {
      return current;
    }
    if (current.data < data) {
      current = current.right;
    }
    if (current.data > data) {
      current = current.left;
    }
  }
  return null;
}
```
查找功能和插入功能是二叉树能力表现的地方了，和二分法一样，时间复杂度为 O(lgn)。
最差情况下是因为二叉树退化成为了链表结构，导致不论查找还是插入功能时间复杂度都为 O(n)。

### 实现其他一些功能
```javascript
// 二叉树查找最小值
getMin() {
  let current = this.root;
  while (current.left !== null) {
    current = current.left;
  }
  return current.data;
}

// 二叉树查找最大值
getMax() {
  let current = this.root;
  while (current.right !== null) {
    current = current.right;
  }
  return current.data;
}

// 求节点个数
getNodeCount() {
  let count = 0;
  this.preOrder(this.root, () => {
    count++;
  })
  return count;
}

// 求边数
getItemCount() {
  let count = 0;
  return this.getNodeCount() - 1;
}
```

最后我们看一下这个二叉树构建出来后的样子。
![](http://7xro5v.com1.z0.glb.clouddn.com/binnaryTree.jpeg)

其实关于二叉树还有很多其他更加高深的，关于二叉树退化成链表的问题也是有优化方案的。

> 参考《数据结构与算法JavaScript描述》
