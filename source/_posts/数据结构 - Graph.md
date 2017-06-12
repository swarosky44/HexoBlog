---
title: 数据结构 - Graph
date: 2017-06-03 15：05
tags: ['数据结构', 'JavaScript']
categories: 数据结构
---

啊 每天都好闲啊 ... 今天来撸图吧

<!-- more -->

## 定义
由顶点与边组成的集合，用于描述顶点的流向的一种数据结构。
图的分类很多, 不过这里我们就不多加分类了。

## 实现
图的实现方式也有很多，常见教材上主要是邻接表的实现，这里一样也采用这种方式。

```javascript
class Graph {
  /**
   * 构造函数
   * @method constructor
   * @param    {Array}     vertices [顶点的集合]
   * @property {Array}     vertices [存储顶点的集合]
   * @property {Object}    adjList  [顶点的邻接表]
   * @property {Object}    marked   [顶点的访问标志]
   * @return   {Graph obj}          [Graph的实例]
   */
  constructor(vertices=[]) {
    this.vertices = vertices;
    this.adjList = {};
    for (let i = 0; i < vertices.length; i++) {
      this.adjList[vertices[i]] = [];
    }
    this.marked = {};
  }

  /**
   * 添加新顶点
   * @method addVertex
   * @param  {String}  v [新的顶点]
   */
  addVertex(v) {
    this.vertices.push(v);
    this.adjList[v] = [];
    this.marked[v] = false;
    return this;
  }

  /**
   * 添加顶点间的边
   * @method addEdge
   * @param  {String} v [顶点]
   * @param  {String} w [顶点]
   */
  addEdge(v, w) {
    this.adjList[v].push(w);
    this.adjList[w].push(v);
    return this;
  }

  toString() {
    let s = '';
    let len = this.vertices.length;
    for (let i = 0; i < len; i++) {
      s += this.vertices[i] + '->';
      let neighbors = this.adjList[this.vertices[i]];
      let nLen = neighbors.length;
      for (let j = 0; j < nLen; j++) {
        s += neighbors[j] + ' ';
      }
      s += '\n';
    }
    console.log(s);
    return this;
  }
}

const graph = new Graph(['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'M', 'N', 'O', 'Z'])
graph
  .addEdge('A', 'B')
  .addEdge('A', 'C')
  .addEdge('A', 'D')
  .addEdge('A', 'G')
  .addEdge('B', 'E')
  .addEdge('B', 'F')
  .addEdge('G', 'H')
  .addEdge('G', 'M')
  .addEdge('G', 'N')
  .addEdge('G', 'O')
  .addEdge('G', 'Z')
  .toString();

/*
 * A->B C D G
 * B->A E F
 * C->A
 * D->A
 * E->B
 * F->B
 * G->A H M N O Z
 * H->G
 * M->G
 * N->G
 * O->G
 * Z->G
 */
```

## 深度优先算法
深度优先（DFS）是图的遍历的一种算法。
其思路很简单：
1. 选定起始顶点，并访问该顶点；
2. 递归访问当前选中顶点的邻接表里面的未访问过的顶点；

eg： 比如我们选择之前的数据
1. 初始顶点定位 A，访问 A；
2. 第一次递归进入 B 顶点，访问 B；
3. 继续递归，访问 B 顶点的邻接表，A 已被访问过，则访问下一个顶点 E。
4. 继续递归，访问 E 顶点的邻接表，A 已被访问过且没有其他邻接顶点，回到上一次递归，访问 B 顶点的邻接顶点 F；
5. 持续递归知道所有顶点都被标记为已访问；

```javascript
dfs(v) {
  this.marked[v] = true;
  if (this.adjList[v] !== undefined) {
    console.log('Visited vertex: ' + v);
  }
  for (let w of this.adjList[v]) {
    if (!this.marked[w]) {
      this.dfs(w);
    }
  }
}
```

## 广度优先算法
深度优先（BFS）是图的遍历的一种算法。
其思路是：
1. 访问初始顶点；
2. 将初始顶点的所有邻接顶点全部入队列；
3. 按队列顺序一个个出列，访问顶点，同时将该顶点的所有未访问的邻接顶点全部入队;
4. 持续迭代，直到队列为空；

eg:
1. 初始顶点定位 A，访问 A；
2. B、C、D、G 作为未访问的邻接顶点入队；
3. 继续迭代，访问 B ，同时 B 的未访问的邻接顶点 E、F 入队；
4. 接续迭代，访问 C，无符合条件的邻接顶点入队；
5. 持续迭代，直到队列为空；

```javascript
bfs(v) {
  const queue = [];
  this.marked[v] = true;
  queue.push(v);
  while (queue.length > 0) {
    const v = queue.shift();
    if (this.adjList[v] !== undefined) {
      console.log('Visited vertex: ' + v);
    }
    for (let w of this.adjList[v]) {
      if (!this.marked[w]) {
        queue.push(w);
        this.marked[w] = true;
      }
    }
  }
}
```

以上就是一个不知道什么分类的简单的图, 当然关于图的高级算法有很多，可是我不会 🙃

> 《 数据结构与算法 JavaScript 描述 》
