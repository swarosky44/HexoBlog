---
title: Canvas - 3D 粒子效果
date: 2017-06-27 15：30
tags: ['Canvas', 'JavaScript']
categories: Canvas
---

最近很喜欢玩 canvas，所以就撸完了[每周一点 canvas 动画](http://supperjet.github.io/tags/canvas/)系列博客以及一本 《 HTML5 Canvas 开发详解 》。

逛了下 codeopen 发现了超多酷炫的 canvas 的作品，就准备自己也来玩一下，做个相似的 3D 粒子效果。😎

<iframe width="100%" height="300" src="//jsfiddle.net/swarosky44/mce99bk2/16/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

<!-- more -->

## 基础知识准备
其实不打算记录太多基础 API 的实现，只是有关几个在这里提一下。
至于那些基础的 API 我推荐到 MDN 上老老实实把 demo 都刷一遍，平常写的时候不记得就立马去查，这些东西，熟能生巧嘛 ~
### 画圆
`ctx.arc(x, y, r, start, end, direction)`

@param [Number]  x：圆心 X 轴坐标；
@param [Number]  y：圆心 Y 轴坐标；
@param [Number]  r：圆半径；
@param [Number]  start：起始弧度值；
@param [Number]  end：结束弧度值；
@param [Boolean] direction：绘制圆的方向， true 为顺时针， false 为逆时针；

<iframe width="100%" height="300" src="//jsfiddle.net/swarosky44/mce99bk2/5/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

### 缩放
`ctx.scale(x, y)`

@param [Number] x：X 轴缩放比；
@param [Number] y：Y 轴缩放比；

这里需要特别说明的事，缩放一个物体，并不是简单减小或放大半径，而是针对 X / Y 轴的刻度缩放。
简单来说就是，如果 0 -> 1 刻度的长度是 1cm, 缩放 x 为 0.5的话，0 -> 1 的长度就是 0.5 cm。也就是说缩放会影响一个物件在我们视觉上的位置感受。

### 透视
既然标题是 3D 粒子，那么当然这里的坐标轴就是不是 2 维的坐标轴了，而是 3 维的坐标轴了，也就是说确定一个物件的位置，需要 (x, y, z) 三个坐标值。

但是如何用 2D 去模拟 3D 呢？推荐看一下这篇文章 -- [3维环境搭建](http://supperjet.github.io/2016/08/17/%E3%80%8A%E6%AF%8F%E5%91%A8%E4%B8%80%E7%82%B9canvas%E5%8A%A8%E7%94%BB%E3%80%8B%E2%80%94%E2%80%943%E7%BB%B4%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

简单总结一下就是，为了在 2D 环境下模拟 3D 运动，需要构建一条 Z 轴。物体在 Z 轴上值的表现在于 scale 的大小。远视则物件越来越小，近视则物件越来越大。

## 步骤
### 一个绘制圆的类
<iframe width="100%" height="300" src="//jsfiddle.net/swarosky44/mce99bk2/15/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

简单介绍一下构造函数里面各个属性:
x：用于绘制的 X 轴的坐标；
y：用于绘制的 Y 轴的坐标；
xp：用于计算的 X 轴的坐标；
yp：用于计算的 Y 轴的坐标；
zp：用于计算的 Z 轴的坐标；
vx：X 轴的速度；
vy: Y 轴的速度；
vz：Z 轴的速度；
r： 球半径；
color： 球填充色值；
scaleX：X 轴缩放比；
scaleY：Y 轴缩放比；
visible： 是否可视；

### 3D 粒子运动
这里是最关键的 --> 如何实现 3D 粒子运动 ?

<iframe width="100%" height="500" src="//jsfiddle.net/swarosky44/mce99bk2/14/embedded/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

#### 1. 准备参数
```JavaScript
const canvas = document.querySelector('canvas');
const ctx = canvas.getContext('2d');
const W = canvas.width = window.innerWidth;
const H = canvas.height = window.innerHeight;
const balls = [];
const ballsNum = 200;
const floor = 100;
const fl = 200;
const g = 0.2;
const bounce = 0.5;
const vpX = W / 2;
const vpY = H / 2;

for (let i = 0; i < ballsNum; i++) {
  const ball = new Ball(5, '#000');
  ball.vx = Math.random() * 10 - 5;
  ball.vy = Math.random() * 10 - 5;
  ball.vz = Math.random() * 10 - 5;
  balls.push(ball);
}
```

这里我们准备了很多参数，一看下去，哇！头皮发麻 ...
balls：用于存储诸多小球的集合；
ballsNum: 小球的上限数目；
floor：我们预设的 y 轴的触底临界值 -> 简单说就是我们在效果图上看到的那个白色玻璃一样的，是假象中的地板。
fl: 我们预设的眼镜到屏幕之间的距离，这个值可以自己随意设置，一般都是 200 ~ 300 之间。
g: 重力加速度。
bounce：弹性系数；
vpX: 消失点的 X 轴坐标；
vpY: 消失点的 Y 轴坐标；

这里关于消失点多逼逼一下，盗用一下一只会飞的鱼的图：
![](https://sfault-image.b0.upaiyun.com/468/049/468049480-57b3deadaec18)
在一个小球越发远离我们的时候，不止球的半径在缩小，连同 X / Y 轴的位置也在发生偏移。

然后我们准备了 200 个小球，同时各自伪随机的赋予了对 x, y, z 三个方向的初始速度。

#### 2.动画循环
```JavaScript
function drawScreen() {
  balls.forEach(move);
  balls.forEach(draw);
}

!function drawFrame() {
  window.requestAnimationFrame(drawFrame);
  ctx.clearRect(0, 0, W, H);
  drawScreen();
}();
```

这里没什么特别的就是每帧动画循环，每帧对每个小球做运动以及绘制的操作；
关于 requestAnimationFrame 的描述，可以到 MDN 上了解一下，简单说就是一个非常人性的定时器，只在浏览器准备好了的情况下才绘制下一帧。
BTW:
`!function drawFrame(){}()` 就是一个自执行函数，和 `(function drawFrame() {})()` 一模一样，单纯省了一个字节，然后好看了点。

#### 3.运动过程
```JavaScript
function move(ball) {
  ball.vy += g;
  ball.xp += ball.vx;
  ball.yp += ball.vy;
  ball.zp += ball.vz;
  if (ball.yp > floor) {
    ball.yp = floor;
    ball.vy *= -bounce;
  }
  if (ball.zp > -fl) {
    const scale = fl / (fl + ball.zp);
    ball.scaleX = ball.scaleY = scale;
    ball.x = vpX + ball.xp * scale;
    ball.y = vpY + ball.yp * scale;
    ball.visible = true;
  } else {
    ball.visible = false;
  }
}

function draw(ball) {
  ball.visible && ball.draw(ctx);
}
```
在`ball.yp > floor`的判断里，我们发现如果小球触底了，则会发生反弹。
在`ball.zp > -fl`的判断里，我们发现小球在 Z 上已经逼近屏幕外我们的眼睛，则不在绘制该小球。
在正常运动中，我们计算小球运动的缩放比 `scale = fl / (fl + ball.zp)` 这里比就是小球到我们眼睛距离的比值，作为缩放值。
当 ball.zp ~ -fl 的时候，scale 是趋近于无限大的。
当 ball.zp ~ 0 的时候，scale 是为 1，认为小球没有缩放。

然后在赋值小球用于绘制的 X / Y 轴坐标里计算`vpX + ball.xp * scale`，这样让小球可以围绕消失点运动，xp 是每帧运动的距离的累加量，vpX 是消失点的 X 轴坐标，所以，小球始终在 vpX 左右运动。
同理 Y 轴坐标。
这样可以防止出现小球飘到左上角的效果问题。

#### 4.小球的绘制
```JavaScript
ctx.save();
ctx.translate(this.x, this.y);
ctx.scale(this.scaleX, this.scaleY);
ctx.fillStyle = this.color;
ctx.beginPath();
ctx.arc(0, 0, this.r, 0, (Math.PI / 180) * 360, false);
ctx.closePath();
ctx.fill();
ctx.restore();
```

这里只有一点需要提醒，为什么要用 translate 做平移，而不是直接绘制小球的时候用 arc 选择好圆心。
因为基础知识里提到了 scale 是对坐标的刻度进行缩放，如果先缩放，再绘制小球则一方面小球的 x, y 坐标在运动，坐标刻度也在运动，这样最终的效果非常差。
所以在缩放之前，先平移定好位值，然后再缩放，最后绘制原点处的小球就不会发生之上的问题。

可以试一试，不用平移的效果，部分会左上角飘，部分会右下角飘。

## 总结
Canvas 还是蛮好玩的，天天撸业务，偶尔做一些酷炫点的东西能很好的保持对前端的兴趣。😀
