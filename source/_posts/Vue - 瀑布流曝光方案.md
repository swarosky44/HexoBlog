---
title: Vue - 瀑布流曝光方案
date: 2018-05-25 10：30
tags:
  - Vue
  - JavaScript
categories: Vue
---

做商品列表肯定会遇见的问题啦 - 瀑布流单品曝光埋点，简单记录一下我的做法。

<!-- more -->

废话不多说，先上代码：

```html
<ul
  class="list-wrapper"
  @scroll="onScrollEvent"
>
  <li
    v-for="d in dataSource"
    :key="d.id"
  >
    <CommodityItem
      ref="commodityItem"
      :dataSource="d"
    />
  </li>
</ul>
```

这是基本的列表结构，我们遍历单品组件 `CommodityItem`，并且给上 ref 标识。

```javascript
export default {
  // ...
  methods: {
    onScrollEvent () {
      this.$refs.commodityItem.forEach(vueComp => {
        vueComp.exposeTrack()
      })
    }
  }
}
```

在列表滚动监听事件里面，我们触发子组件里面埋点事件。这里通过 refs 可以获取 ref 标识为 commodityItem 的全部子组件：`this.$refs.commodityItem = [vueComp, vueComp, vueComp, ...]`


```javascript
export default {
  data () {
    return {
      hasExposed: false
    }
  },

  methods: {
    exposeTrack () {
      const rect = this.$el.getBoundingClientRect()
      if (rect.top < window.innerHeight && rect.bottom > 0 && !this.hasExposed) {
        this.$track.ec()
        this.hasExposed = true
      }
    }
  }
}
```

在子组件里面我们定义触发埋点的事件，通过父组件来调用。用 `getBoundingClientRect` 的 API 来判断当前该子组件是否在视图之中，通过 `hasExposed` 来避免二次曝光。

以上就是我实现瀑布流滚动单品曝光的方案，总体来说精确度还不错。
