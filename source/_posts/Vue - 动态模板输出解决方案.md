---
title: Vue - 动态模板输出解决方案
date: 2018-02-06 15：30
tags:
  - Vue
  - JavaScript
categories: Vue
---

又到年终啦，最近接手的需求中遇见一个动态模板输出优化的问题，想了一个自觉很 easy 的解决方案，过来记录一下。

<!-- more -->

## 问题描述
需求是实现一个 App 内嵌 H5 用于显示文章详情页面。

这里比较麻烦的点是 - 由于文章内容由编辑后台的运营人员输出，前端通过 API 得到的是一串 HTML 源码的字符串。但是里面会穿插许多自定义标签，前端需要更具设计图去动态实现自定义标签里面的内容和事件，以及异步请求自定义标签内所需数据。
API 返回数据举例如下：
```JSON
{
  "modified_html_info": "<h1 class="style-scope wysiwyg-e">素人穿搭Vol.3 她们一周5天上班都穿得啥？</h1><hmy-image-to-collocation obj-id="27590" src="http://hmy-user.image.alimmdn.com/article-editor-photo/2018-1-30/3999154/1517290259511-8F32216A13AD897A3EC6.jpg@710w.jpg" type="hmy-element"></hmy-image-to-collocation><p>关注 charming 噢</p>"

}
```

这里我可能需要将 `<hmy-image-to-collocation></hmy-image-to-collocation>` 输出成一个特定设计的组件，列如一个轮播组件或者一个 Canvas 输出虚拟形象的组件。

## 优化方案
之前的方案：
- 通过字符串模板的正则匹配，替换 `<hmy-image-to-collocation></hmy-image-to-collocation>` 标签，将组件内容也写成字符串完成替换，然后通过 `v-html` 输出。异步请求数据和自定义事件需要通过直接操作 DOM 完成。

优化方案：
- vue 的动态组件 `is` 属性 + Vue 的局部组件。

## 实现步骤
1. 在页面初始化时请求文章内容，得到 `modified_html_info` 字符串
```javascript
export default {
  ...,
  methods = {
    async fetchArticle () {
      cosnt pk = this.$route.params.pk
      data = await api.article.getArticle(pk)
    }
  }
}
```
2. 通过 `Vue.extend` 创建一个 Vue 的组件实例，`modified_html_info` 作为 template 的内容
```javascript
export default {
  ...,
  methods = {
    async fetchArticle () {
      cosnt pk = this.$route.params.pk
      data = await api.article.getArticle(pk)
      cosnt Article = {
        template: template: `<div>${data.modified_html_info}</div>`
      }
    }
  }
}
```
3. 根据设计 & 产品需求，将可能出现的自定义标签都按组件实现一遍。
```javascript
import HmyImageToCollocation from './HmyImageToCollocation'
import HmyImageToSku from './HmyImageToSku'
import HmyMedelToCollocation from './HmyMedelToCollocation'
import HmyMedelToSku from './HmyMedelToSku'
import HmySwipeToCollocation from './HmySwipeToCollocation'

export default {...}
```
4. 在 我们创建的 Vue 的局部组件中完成这些自定义组建的注册。
```javascript
export default {
  ...,
  methods = {
    async fetchArticle () {
      ...
      const Article = {
        template: `<div>${data.modified_html_info}</div>`,
        components: {
          HmyImageToCollocation,
          HmyImageToSku,
          HmyMedelToCollocation,
          HmyMedelToSku,
          HmySwipeToCollocation
        }
      }
    }
  }
}
```
因为返回的模板字符串不一定会有一个根标签，所以我们最好自己添加上一个根标签。
5. 将我们创建的局部组件实例，通过 is 特性注入到当前的页面中
```javascript
  <template>
    <component class="article-wrapper" :is="currentView" />
  </template>

  <script>
    export default {
      data () {
        return {
          currentView: null
        }
      },
      methods: {
        ...
        const Article = {...}
        this.currentView = Article
      }
    }
  </script>
```
6. 在自定标签的组件内获取自定义属性上的值（可能用于计算或者请求）
```javascript
// HmyImageToSku
export default {
  created () {
    const { obj-id: id } = this.$vnode.data.attrs
    this.id = id
  }
}
```

以上就是完成这种混杂自定标签的字符串模板的解决方案，避免过多的字符串操作以及 DOM 操作，通过组件思想来完成。
