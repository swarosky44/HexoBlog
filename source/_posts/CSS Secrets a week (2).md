---
title: CSS Secrets a week
date: 2017-04-10 10:48
tags: CSS
categories: CSS
---

偷个闲，写一篇小短文关于用 CSS 画平行四边形的两种方法。

<!-- more -->

## 难题：平行四边形

试着用 CSS 创建一个按钮状的平行四边形，通常我们都会用 `skew()` 变形属性来实现。

```css
.button {
  width: 200px;
  height: 50px;
  border: 1px solid blue;
  transform: skew(-45deg);
}
```

这样的确能够是一个元素实现变形，向右倾斜 45 度角。但是元素内的其他内容也会跟随变形。

![](http://oe3rjg2g6.bkt.clouddn.com/skew.png)

我们需要解决的也就是这个难题。

## 嵌套元素解决

因为外层元素发生了倾斜，我们可以让容器再多包裹一层元素，反向倾斜来解决这个问题。

```html
<div class="button">
  <p class="text">
    hello world
  </p>
</div>
```

```css
.button {
  width: 200px;
  height: 50px;
  border: 1px solid blue;
  transform: skew(-45deg);
}
.button .text {
  text-align: center;
  transform: skew(45deg);
}
```

这个方法显然是有效的。

![](http://oe3rjg2g6.bkt.clouddn.com/skew%282%29.png)

但是如果内层元素太多，就需要一个个去反向变形，显得很冗余。

## 伪元素方案

这个方案在于在背景、边框、变形等样式应用到伪元素上，这样容器内的元素是不会发生变形的。

```css
.button {
  width: 200px;
  heigth: 200px;
  position: relative;
}
.button::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  border: 1px solid blue;
  transform: skew(45deg);
  z-index: -1;
}
.button .text {
  text-align: center;
}
```

