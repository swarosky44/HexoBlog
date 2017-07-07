---
title: Canvas - 3D 旋转
date: 2017-07-07 11：02
tags: ['Canvas', 'JavaScript']
categories: Canvas
---

最近两天撸了两个 canvas，3D 旋转 & ribbon。

<iframe width="100%" height="300" src="//jsfiddle.net/eLnpwuye/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

3D 旋转我是从[这里](http://supperjet.github.io/2016/09/29/%E3%80%8A%E6%AF%8F%E5%91%A8%E4%B8%80%E7%82%B9canvas%E5%8A%A8%E7%94%BB%E3%80%8B%E2%80%94%E2%80%943D%E6%97%8B%E8%BD%AC%E4%B8%8E%E7%A2%B0%E6%92%9E/)学来的，因为原博主写的比较简单，所以重新实现了一下。

<!-- more -->

## 基础知识
- [3维模型构建](http://supperjet.github.io/2016/08/17/%E3%80%8A%E6%AF%8F%E5%91%A8%E4%B8%80%E7%82%B9canvas%E5%8A%A8%E7%94%BB%E3%80%8B%E2%80%94%E2%80%943%E7%BB%B4%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/) & [3D粒子效果](http://swarosky44.github.io/2017/06/27/Canvas%20-%203D%20%E7%B2%92%E5%AD%90%E6%95%88%E6%9E%9C/#more)
- [圆周运动](http://supperjet.github.io/2016/07/06/%E3%80%8A%E6%AF%8F%E5%91%A8%E4%B8%80%E7%82%B9canvas%E5%8A%A8%E7%94%BB%E3%80%8B%E2%80%94%E2%80%94%E5%9D%90%E6%A0%87%E6%97%8B%E8%BD%AC/)

## 基本原理
基本了解了圆周运动的坐标计算公式 & 3D模型之后，简单总结一下就是，如果想模拟一个物体的 3D 旋转，就必须扩展一下圆周运动的坐标计算公式。
盗用一下一只会飞的鱼的图：
![](https://sfault-image.b0.upaiyun.com/362/072/3620720564-57de449b83024_articlex)

由此可以得到公式
```JavaScript
// 绕Z轴
newX = x * cos(angleZ) - y * sin(angleZ);
newY = y * cos(angleZ) + x * sin(angleZ);

// 绕X轴
newY = y * cos(angleX) - z * sin(angleX);
newZ = z * cos(angleX) - y * sin(angleX);

// 绕Y轴
newX = x * cos(angleY) - z * sin(angleY);
newZ = z * cos(angleY) + x * sin(angleY);
```

实际操作上，我们是没有办法改变鼠标在 Z 轴上的值，我们只能改变 X 和 Y 轴的坐标，因此主要的两个旋转公式就是围绕着 X 和 Y 使用的。

## 实现
### 参数准备
```JavaScript
const W = canvas.width = window.innerWidth;
const H = canvas.height = window.innerHeight;
const balls = [];
const ballsNum = 250;
const fl = 250;
const vpX = W / 2;
const vpY = H / 2;
let mouseX = 0;
let mouseY = 0;
let angleX;
let angleY;
```
- W 、H: canvas 的宽高；
- balls: 用来存储小球的容器；
- ballsNum: 小球数量的上限；
- fl: 眼睛距离屏幕的距离；
- vpX、vpY: 消失点的坐标，即 canvas 的中心点；
- mouseX、mouseY: 鼠标的 X 、Y 轴坐标；
- angleX、angleY: X 和 Y 轴的旋转角度；

### 生成小球
```JavaScript
for (let i = 0; i < ballsNum; i++) {
  const r = Math.random() * 5 + 1;
  const ball = new Ball(r);
  ball.xpos = Math.random() * 300 - 150;
  ball.ypos = Math.random() * 300 - 150;
  ball.zpos = Math.random() * 300 - 150;
  balls.push(ball);
}
```
这里很简单，随机定义小球的半径 & x、y、z 轴的坐标，并且推入存储容器中。

### 旋转角度计算
```JavaScript
angleX = (mouseY - vpY) * 0.0001;
angleY = (mouseX - vpX) * 0.0001;
```

这里我们用鼠标位置与消失点的偏移量来计算 X 、 Y 轴的旋转角度。

### 运动量计算
```JavaScript
function move(ball) {
  rotateX(ball, angleX);
  rotateY(ball, angleY);
  setPerspective(ball);
}

function rotateX(ball, angle) {
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);
  const y1 = ball.ypos * cos - ball.zpos * sin;
  const z1 = ball.zpos * cos + ball.ypos * sin;
  ball.ypos = y1;
  ball.zpos = z1;
}

function rotateY(ball, angle) {
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);
  const x1 = ball.xpos * cos - ball.zpos * sin;
  const z1 = ball.zpos * cos + ball.xpos * sin;
  ball.xpos = x1;
  ball.zpos = z1;
}

function setPerspective(ball) {
  if (ball.zpos > -fl) {
    ball.scaleX = ball.scaleY = fl / (fl + ball.zpos);
    ball.x = vpX + ball.xpos * ball.scaleX;
    ball.y = vpY + ball.ypos * ball.scaleY;
    ball.visible = true;
  } else {
    ball.visible = false;
  }
}
```

这里就是对之前原理的一个运用，计算 X 、 Y 的轴旋转后的各个小球的坐标。
最后再计算小球在 Z 轴的坐标对缩放的影响。

## 总结
其实只要掌握了坐标计算公式 & 3D 环境搭建的原理，3D 旋转其实还是蛮容易的，但是做出来的效果却非常酷炫，到 codeopen 上可以看到各种各样的旋转效果。
所以之前提及的几篇文章可以多看看，那些是非常基础的概念，理解好之后再加上点创意，就可以创造很多了 🤓
