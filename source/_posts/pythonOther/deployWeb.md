
---
title: 花最小的学习成本实现web UI
date: 2021-07-05 11:51:02
tags:
categories: 工具人
---

- [给程序搭个web界面](#%E7%BB%99%E7%A8%8B%E5%BA%8F%E6%90%AD%E4%B8%AAweb%E7%95%8C%E9%9D%A2)
    - [预备知识](#%E9%A2%84%E5%A4%87%E7%9F%A5%E8%AF%86)
    - [总体思路](#%E6%80%BB%E4%BD%93%E6%80%9D%E8%B7%AF)
        - [1.整体思路](#1-%E6%95%B4%E4%BD%93%E6%80%9D%E8%B7%AF)
        - [2.面临的困难](#2-%E9%9D%A2%E4%B8%B4%E7%9A%84%E5%9B%B0%E9%9A%BE)
        - [3.前端框架的选择](#3-%E5%89%8D%E7%AB%AF%E6%A1%86%E6%9E%B6%E7%9A%84%E9%80%89%E6%8B%A9)
    - [实现过程](#%E5%AE%9E%E7%8E%B0%E8%BF%87%E7%A8%8B)
        - [1.快速搞定布局问题](#1-%E5%BF%AB%E9%80%9F%E6%90%9E%E5%AE%9A%E5%B8%83%E5%B1%80%E9%97%AE%E9%A2%98)
        - [2.数据与组件绑定](#2-%E6%95%B0%E6%8D%AE%E4%B8%8E%E7%BB%84%E4%BB%B6%E7%BB%91%E5%AE%9A)
        - [3.动态页面渲染](#3-%E5%8A%A8%E6%80%81%E9%A1%B5%E9%9D%A2%E6%B8%B2%E6%9F%93)
        - [4.协议交互](#4-%E5%8D%8F%E8%AE%AE%E4%BA%A4%E4%BA%92)
        - [5.事件响应](#5-%E4%BA%8B%E4%BB%B6%E5%93%8D%E5%BA%94)
    - [总结](#%E6%80%BB%E7%BB%93)


# 给程序搭个web界面
作为一名合格的工具人，通过web界面/客户端来组织管理你写的工具是必不可少的技能，否则你的工具是不可能得到其他同事的认同的。
但我们毕竟不是专门搞前端的，因此有了这篇学习记录，以备下次再遇到这种事情能快速响应。

### 预备知识
这一次我们主要聊聊前端的预备知识，而C端和服务端下次再聊：
1. 了解html、css、js，其中html和js可以在实际开发过程中边学边用
2. 了解[vue](https://cn.vuejs.org/guide/introduction.html)，我觉得只需要知道"模板语法"就可以上手了，后面一样边学边用
3. 了解[element](https://element.eleme.cn/#/zh-CN/component/layout)，文档非常的丰富，用到哪个组件就去看对应的文档即可
4. 了解http协议，主要是出问题的时候，可以抓包快速确定是前端还是后端的问题

### 总体思路
搭建web这件事，在现在这家公司我搞了2次，第一次是我们工具人刚起步的时候，第二次则是1年半之后对第一次的推翻重构。

##### 1.整体思路
虽然玩了2次，但整体思路是一致的：
1. 先确定要做成什么样子，然后从网上找一个静态页面模板 
2. 确定前后端的交互协议,确定动态数据
3. 结合数据，将这个静态页面改造成动态页面
4. 部署服务器，实现后端脚本

##### 2.面临的困难
但对于一个前端或C端的小白来说，我相信我们面临的困难是一样的：
1. 页面的布局问题(大到整个页面的布局，小到一行、一列的布局)
2. 组件的样式问题(看见CSS就痛疼)
3. 数据与页面绑定的问题(数据与界面该如何分离)
4. 组件的事件响应(第一玩的时候，我居然把导航栏的切换给实现了)
5. 前后端的通信问题

##### 3.前端框架的选择
对于前面的这些小白问题，如果你用原始的html+css+js来开发，会异常的艰苦。
像我第一次采用了bootstrap来实现，当时为了实现数据与UI的绑定，写了大量的if-else，而如果想要简单的扩展一个功能，往往要花上1天时间。
因为我没有前端的知识基础，我的页面是从[bootstrap的官网](https://getbootstrap.com/docs/5.0/getting-started/download/)找来的模板来实现的。
这虽然节省了一些时间，但后面很难扩展。
而第二次开发则采用Vue+element来开发，简单来说这就是目前最流行的前端框架之一，换了一种框架之后，像上述的困难点2、3 基本不用考虑了。

### 实现过程
##### 1.快速搞定布局问题
由于我们不熟悉也不打算学习传统的\<div\> 等html标签，因此我们可以通过[拖拽生成web UI](https://vcc3.sahadev.tech/)。有了这个玩意之后，我就可以任意的搭建我们的UI了。
通过拖拽，轻松搭建完成下面的静态网页：
![](Images/vcc3_web.png)
当然玩过几次之后，你会发现这玩意也挺不好用的，它仅仅用来帮你搭建大致的代码框架。

##### 2.数据与组件绑定
在搭建完页面的框架之后，我们用到的组件还是默认的格式与数据，下面我们需要通过[element](https://element.eleme.cn/#/zh-CN/component/layout)的官方文档，找到对应组件的用法。
比如输入框组件与数据的绑定:
```html
<!-- 来自element官网的例子 -->
<el-input
  placeholder="请输入内容"
  v-model="input"
  clearable>
</el-input>

<script>
  export default {
    data() {
      return {
        input: ''
      }
    }
  }
</script>
```

这里的v-model指令就是用于实现双向数据绑定的。
知道这个功能之后，我们就可以开始设计我们的数据结构了，因为不同的组件要求绑定的数据格式可能是不一样的，比如能多选的组件可能要求数组格式。

##### 3.动态页面渲染
接下来，我们希望通过后端返回的数据来动态展示web的界面，同样通过Vue的指令，我们就可以实现对组件的展示做动态处理，如：
```html
<!-- 自己瞎写的例子 -->
<template v-for="item in business">
    <template v-if='item["value"].length >= 6'>
        <el-input type="textarea" v-model='item["value"]' :disabled='!item["is_enable"]'>
        </el-input>
    <template v-else>
        <el-input v-model='item["value"]' :disabled='!item["is_enable"]'>
        </el-input>
</template

<script>
  export default {
    data() {
      return {
        business: [
          {
            is_enable: false,
            value: "随手写6个字",
          },
          {
            is_enable: true,
            value: "2字",
          },
        ]
      }
    }
  }
</script>
```
这个例子，通过后端返回的business数据，动态的展示成多个input输入框，而且还可以根据数据的长度动态决定input还是textarea输入框。

##### 4.协议交互
结合前面UI展示所要用到的数据，我们现在需要从服务器取数据，这里推荐使用axios来完成请求。
```js
const app = {
  data() {
    return {
      language: null,
    }
  },
  mounted () {
    axios
      .get('/cgi-bin/test/loadLanguage.cgi')
      .then((response) => {
        this.language = response.data
      })
      .catch(function (error) { // 请求失败处理
        console.log(error);
      });
  }
}
Vue.createApp(app).use(ElementPlus).mount("#app");
```
这里会出现的问题可能就比较多了，比如http协议中"Content-type"设置不正确，表单数据可能要用 window.Qs.stringify(params)格式化等等

##### 5.事件响应
组件的事件响应，还是一样通过[vue](https://cn.vuejs.org/guide/essentials/event-handling.html#listening-to-events)以及[element](https://element.eleme.cn/#/zh-CN/component/layout)的文档来确定。
```html
<el-button @click="open">点击</el-button>
<script>
  export default {
    methods: {
      open() {
        this.$message('这是一条消息提示');
      },
    }
  }
</script>
```
这部分不难，但有时候也会遇到奇葩问题，比如用Cascader组件，通过blur事件来实现自动保存数据的功能，但直到最后也没解决。

### 总结
最后贴一张最终的效果图：
![](Images\vue_web.png)
我想，界面的好差可能关乎到领导对你做的事情的初步印象，而通过Vue+element应该能做到高端的视觉体验，而且开发成本不大。

