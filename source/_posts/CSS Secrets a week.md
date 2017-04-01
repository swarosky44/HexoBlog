---
title: CSS Secrets a week
date: 2017-03-25 17:51
tags: CSS
categories: CSS
---
很久没有写文章了，忙乎了大半年的项目 4 月 1 号就要上线了。相对之前个人时间也充裕了起来，继续写一篇关于 CSS Secrets 的二刷笔记吧 😎

在日常开发中我们会遇见需要用 CSS 绘制一个圆的需求，通常来说我们需要元素等高等宽，然后将 `border-radius` 设置为大于等于高度（宽度）的一半的值。但是我们是否真的理解了 `border-radius` 属性的真实含义？

<!-- more -->

## 难题：自适应的椭圆

日常我们利用 CSS 绘制一个圆，代码如下：

```css
.circle {
  width: 200px;
  height: 200px;
  border: 1px solid red;
  border-radius: 100px;
}
```

如果元素并非等高等宽，则将绘制出一个椭圆。

这里就需要我们对 `border-radius` 属性深入了解一下，我们可以利用 `border-radius` 来实现一些怎样形状的元素。

在 [MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/border-radius) 中描述了 `border-radius` 这个属性的值得具体描述：

```css
border-radius: <length-percentage>{1, 4} [/ <length-percentage{1, 4}>]
```

`length-percentage`  - 可以为百分比或者具体的像素值（px），指代圆角在一个方向上的值，百分比的值是依据		元素的最终宽高值计算得到。

`{1, 4}` - 必须制定至少一个最多四个角的圆角值；

`[/<length-percentage{1, 4}>]` - 如果设置了该值则代表对圆角在两个方向上的值分别设置；

这里我们我可以发现 `border-radius` 身上有两个惊天大秘密 😱

#### `border-radius` 其实是有两个方向上的值可以设置 - 水平方向 & 垂直方向

![](http://oe3rjg2g6.bkt.clouddn.com/border-radius.jpeg)



从上面的图中我们可以看到一个圆角的构成，是基于 X 轴& Y 轴的两个方向上构成的，如果 X 轴 & Y 轴的值一样，可以缩写成一个，这个时候会得到一个宽高按比例收缩的圆的四分之一圆弧。

当然我们也可以对两个方向上的值分别设置不同的值，eg:

```css
.circle {
  width: 200px;
  height: 200px;
  border: 1px solid red;
  border-radius: 20% / 40%;
}
```

这个时候我们得到的元素明显在 Y 轴的弧度要大于 X 轴的弧度。

![](http://oe3rjg2g6.bkt.clouddn.com/border-radius2.png)



需要特别说明的是在写 less 的时候，这类预处理语言需要将 '/' 经过转义才可以使用

```less
border-radius: 20% ~'/' 40%;
```



#### `border-radius` 其实是一个缩写属性

在`border-radius`语法中我们看到其实可以分别设置 4 个角的值

```css
border-radius: 10% 20% 5px 0 / 40% 20% 10px 80%;
```

这样一个 `border-radius` 属性可以解读成：

左上角 - 10% 的水平半径，40% 的垂直半径；

右上角 - 20% 的水平半径，20% 的垂直半径；

右下角 - 5px 的水平半径，10px 的垂直半径；

左下角 - 没有水平半径，80% 的垂直半径；

所以这里可以展开写成：

```css
border-top-left-radius: 10% / 40%;
border-top-right-radius: 20% / 20%;
border-bottom-right-radius: 5px / 10px;
border-bottom-left-radius: 0 / 80%;
```

和所有缩写属性一样，如果 `border-radius` 只有两个角的值话，那么前者代表左上角 & 右下角，而后者代表右上角 & 左下角，我们同样可以记忆为 上右下左 的顺序。

彻底理解了这两个属性之后我们可以做很多有趣的形状，比如半圆、半椭圆、四分之椭圆之类。



***



> [border-radius 属性详解](https://developer.mozilla.org/zh-CN/docs/Web/CSS/border-radius)