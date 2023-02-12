---
title: 解决在ios上小程序webview弹窗滚动卡顿问题
tags:
  - coding
abbrlink: 3dbe7413
date: 2023-01-28 15:59:22
---
微信小程序webview弹窗滚动卡顿问题，这个问题在ios上比较明显，安卓上丝滑流畅，不清楚原因。
<!--more-->
## 问题描述
究竟有多卡，看视频：

{% dplayer "url=https://hui.zone/post/3dbe7413/127_1674892516.mp4" %}

## 问题解决
overflow: scroll 写得好好的，安卓机和Edge浏览器没有问题，排除代码低级错误。
测试的时候稍微多滑了一下，发现整个页面都滑动了，显示出来微信小程序的显示网址功能，感觉这里可能是问题所在。
立即推，在网上找了一些禁止小程序上下滑动的资料，说什么阻止touchmove事件、给app.json配置disableScroll属性，都不管用。
最终将弹窗底下的场景元素设置为fixed，问题解决。

效果：
{% dplayer "url=https://hui.zone/post/3dbe7413/128_1674892518.mp4" %} 
