---
title: 数据结构 - AVL Tree
date: 2017-06-01 15：05
tags: ['数据结构', 'JavaScript']
categories: 数据结构
---

今天我又来撸 🌲 了...

<!-- more -->

## 平衡二叉树
AVL树并不是一种特殊的树结构，它也是二叉查找树（Binary Tree）的一种实现方式。
之前有提到过二叉查找树在查找功能和插入功能上时间复杂度为 O(lg2);
但是最差情况下会退化成链表结构，因此时间复杂度为 O(n);
因此在二叉查找树的基础上有人提出了优化方案 - 平衡二叉树。

## 定义
平衡二叉树在二叉查找树的实现上多了一条规则 - 它的左右两个子树的高度差的绝对值不能超过1。
而今天我们要实现的 AVL树正式平衡二叉树的一种实现方式。实际上实现方式并不单一，还包括红黑树、替罪羊树、伸展树 ...反正我都不会 🙃 。

## 步骤
为了防止直接撸码会逻辑混乱 ... 我先试着梳理一下整个实现过程的步骤：
1. 递归查找插入新节点的对应位置 ;
2. 新节点插入后，判断当前树结构是否平衡（左右子树高度差的绝对值不能小于 1）;
3. 如果平衡那就没事了 ...
4. 如果不平衡就进行旋转，使二叉树依然保持平衡状态；

以上就是向一棵AVL树插入节点的过程。

## 实现
首先我们可以创建一个新类，大部分地方是可以继承之前写的 Binary Tree 的，只需要添加几个方法，同时修改一下 insert 方法即可。

```javascript
class Node {
  constructor(data, left = null, right = null) {
    this.data = data;
    this.left = left;
    this.right = right;
  }
}

class AVLTree extends BTS {
  constructor() {
    super();
  }

  insert() {}

  insertNode() {}

  isBalance() {}

  getNodeHeight() {}

  rotate() {}

  findBeforeNode() {}
}
```

### 实现插入
```javascript
insert(data) {
  const node = new Node(data);
  if (this.root === null) {
    this.root = node;
  } else {
    this.insertNode(this.root, node);
  }
}

insertNode(node, newNode, direction) {
  if (node.data < newNode.data) {
    if (node.right === null) {
      node.right = newNode;
      const balanceNode = this.isBalance(this.root);
      if (balanceNode) {
        console.log(balanceNode, '节点 -- 不平衡');
        this.rotate(balanceNode, direction);
      }
    } else {
      const direct = direction || 'right';
      this.insertNode(node.right, newNode, direct);
    }
  } else if (node.data > newNode.data) {
    if (node.left === null) {
      node.left = newNode;
      const balanceNode = this.isBalance(this.root);
      if (balanceNode) {
        console.log(balanceNode, '节点 -- 不平衡');
        this.rotate(balanceNode, direction);
      }
    } else {
      const direct = direction || 'left';
      this.insertNode(node.left, newNode, direct);
    }
  } else {
    throw new Error('树中已存在该值, 不允许重复存储！');
    return;
  }
}
```
我们将原来一个 insert 的函数拆分成了两个函数，然后递归调用 insertNode 函数，一步一步找到适合 data 存储的节点位置。
其实大部分代码和之前的是差不多的，两个转变：
1. 迭代变成了递归；
2. 插入了新节点之后多了一段验证树结构是否平衡的代码。通过两个函数 isBalance & rotate 实现了树的自平衡功能。

### 实现验证树是否平衡
```javascript
isBalance(node) {
  const leftTree = this.getNodeHeight(node.left);
  const rightTree = this.getNodeHeight(node.right);
  const remainder = leftTree - rightTree;
  if (remainder === -2) {
    console.log('右子树不平衡');
    return node;
  } else if (remainder === 2) {
    console.log('左子树不平衡');
    return node;
  } else {
    const balanceLeft = node.left ? this.isBalance(node.left) : null;
    const balanceRight = node.right ? this.isBalance(node.right) : null;

    if (balanceLeft) {
      return balanceLeft;
    } else if (balanceRight) {
      return balanceRight;
    } else {
      return null;
    }
  }
}

getNodeHeight(node) {
  if (node === null) {
    return 0;
  }
  const oLeft = this.getNodeHeight(node.left);
  const oRight = this.getNodeHeight(node.right);
  return 1 + Math.max(oLeft, oRight);
}
```
isBalance 用于验证当前传入节点 node 的左子树 & 右子树之间的层级差的是否超过了 2 层, 超过了就返回子树不平衡的父节点，用于 rotate 处理该父节点以及以下的子树。
这段函数里面：
1. 先求得当前节点的左子树深度 & 右子树的深度。
2. 如果 |深度差| === 2， 我们就返回当前的节点。这里为什么是 === 而不是 >= 呢？因为 2 是异常的临界值，每次插入新节点的时候一旦发生不平衡的情况就会被旋转，所以子树之间的层级差最大值也就是 2 层。
3. 如果 |深度差| < 2, 我们就继续递归当前节点的子节点，直到发现不平衡的存在。
4. 如果树是平衡的，则返回 null。

getNodeHeight 是一个工具函数。
用于返回当前节点下，子树的深度，通过递归返回从当前父节点，到子树最下面的节点之间的路径长，然后累加返回，最后取左子树 & 右子树之间最大值返回。

### 实现自平衡（旋转）
```javascript
rotate(balanceNode, direction) {
  let nextNode = balanceNode[direction];
  let nextNodeLeft = nextNode.left;
  let nextNodeRight = nextNode.right;
  // ...
}
```

需要旋转的情况比较多，以下只写左子树的情况，因为右子树只需要与其相反就行了（即左子树需要右旋转，那么右子树就需要做旋转）。
顺便借用一下慕课网的图 ...🙂

#### 第一种情况
![](http://7xro5v.com1.z0.glb.clouddn.com/avl1.jpeg)

```javascript
if (direction === 'left') {
  if (nextNode.right === null) {
    nextNode.right === balanceNode;
    balanceNode.left === null;
  }
}
```

#### 第二种情况
![](http://7xro5v.com1.z0.glb.clouddn.com/avl2.jpeg)

```javascript
if (direction === 'left') {
  if (nextNode.right !== null) {
    balanceNode.left = nextNode.right;
    nextNode.right = balanceNode;
  }
}
```

#### 第三种情况
![](http://7xro5v.com1.z0.glb.clouddn.com/avl3.jpeg)

```javascript
if (direction === 'left') {
  if (!nextNodeLeft && nextNodeRight) {
    nextNode.right = null;
    balanceNode.left = nextNode.right;
    nextNodeRight.left = nextNode;
  }
  if (nextNode.right === null) {
    nextNode.right === balanceNode;
    balanceNode.left === null;
  }
}
```

#### 第四种情况
![](http://7xro5v.com1.z0.glb.clouddn.com/avl4.jpeg)

```javascript
if (direction === 'left') {
  if (nextNodeLeft && nextNodeRight && this.getNodeHeight(nextNodeLeft) < this.getNodeHeight(nextNodeRight)) {
    nextNode.right = null;
    balanceNode.left = nextNodeRight;
    nextNodeRight.left = nextNode;
  }
  if (nextNode.right !== null) {
    balanceNode.left = nextNode.right;
    nextNode.right = balanceNode;
  }
}
```

#### 完整代码
```javascript
rotate(balanceNode, direction) {
  let nextNode = balanceNode[direction];
  let nextNodeLeft = nextNode.left;
  let nextNodeRight = nextNode.right;

  if (direction === 'right') {
    if (nextNodeLeft && !nextNodeRight) {
      nextNode.left = null;
      balanceNode.right = nextNodeLeft;
      nextNodeLeft.right = nextNode;
    } else if (nextNodeLeft && nextNodeRight && this.getNodeHeight(nextNodeLeft) > this.getNodeHeight(nextNodeRight)) {
      nextNode.left = null;
      balanceNode.right = nextNodeLeft;
      nextNodeRight.right = nextNode;
    }
    nextNode = balanceNode[direction];
    if (nextNode.left === null) {
      nextNode.left = balanceNode;
      balanceNode.right = null;
    } else {
      balanceNode.right = nextNode.left;
      nextNode.left = balanceNode;
    }
  } else if (direction === 'left') {
    if (!nextNodeLeft && nextNodeRight) {
      nextNode.right = null;
      balanceNode.left = nextNode.right;
      nextNodeRight.left = nextNode;
    } else if (nextNodeLeft && nextNodeRight && this.getNodeHeight(nextNodeLeft) < this.getNodeHeight(nextNodeRight)) {
      nextNode.right = null;
      balanceNode.left = nextNodeRight;
      nextNodeRight.left = nextNode;
    }
    nextNode = balanceNode[direction];
    if (nextNode.right === null) {
      nextNode.right = balanceNode;
      balanceNode.left = null;
    } else {
      balanceNode.left = nextNode.right;
      nextNode.right = balanceNode;
    }
  }

  if (this.root === balanceNode) {
    this.root = nextNode;
  } else {
    const beforeNode = this.findBeforeNode(this.root, balanceNode);
    beforeNode[direction] = nextNode;
  }

  const newBalanceNode = this.isBalance(this.root);
  if (newBalanceNode) {
    console.log(newBalanceNode, '节点第二次 -- 不平衡');
    this.rotate(newBalanceNode, direction);
  }
}

findBeforeNode(root, node) {
  if (root.left === node || root.right === node) {
    return root;
  } else {
    if (root.left) {
      const resultL = this.findBeforeNode(root.left, node);
      if (resultL !== null) {
        return resultL;
      }
    }
    if (root.right) {
      const resultR = this.findBeforeNode(root.right, node);
      if (resultR !== null) {
        return resultR;
      }
    }
    return null;
  }
}
```

综上就完成了AVL树的全部内容了 ... 其实对于多种情况下树的旋转如果不是很明白，简单画一下草图，然后跟着代码旋转两次就容易明白多了。

> 参考 [AVL平衡二叉树 AND 旋转 js 版](http://www.imooc.com/article/16589?block_id=tuijian_wz)
