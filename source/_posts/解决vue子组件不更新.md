---
title: 解决vue子组件不更新
abbrlink: 5dc374b5
date: 2022-08-01 16:40:59
tags:
- vue
---

又遇到了令人头疼的不更新问题，这次我没有强制渲染（反模式），换了一种贴合逻辑的方法解决。

<!--more-->

## 问题描述

看一下我的问题

<video muted="muted" autoplay="autoplay" loop="loop" playsinline="" class="demo-video" style="width:100%"><source src="/post/5dc374b5/QQ20220801-164654.mp4" type="video/webm"> </video>

这里明显可以看到，两个选择题切换过后就不更新了，怎么点都是之前那个，实际应该更新成路由上面对应id的题目。

```html
<template>
  <ChoiceQ :q="Q" :roomId="roomId" v-if="choiceQShow"/>
  <FillblankQ :q="Q" :roomId="roomId" v-if="fillBlankShow"/>
</template>
```



## 解决过程

起初考虑是路由问题，可能是选择题组件写的有问题？后来发现都是可以监听到`props`更新的。

折磨了一下午，猜测可能是更新的时候渲染函数误认为两个选择题组件是同一个组件，所以不更新？那我区分一下，想到了`key`，试了一下果然可以。

在源代码基础上添加key属性

```html
<template>
  <ChoiceQ :q="Q" :roomId="roomId" v-if="choiceQShow" :key="Q.id"/>
  <FillblankQ :q="Q" :roomId="roomId" v-if="fillBlankShow" :key="Q.id"/>
</template>
```

当`Q.id`发生变化之后，vue 便可以根据`Q.id`区分是哪个组件更新了，从而触发更新。

后来查资料发现`key`这个属性还真不能小看它！



详细资料：[如何理解vue中的key？ - 掘金 (juejin.cn)](https://juejin.cn/post/6844903985397104648)
