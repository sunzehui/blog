---
title: 解决echarts在高分屏下高度异常问题
date: 2023-04-28 17:14:07
tags:
- coding
---

一直没在乎过window系统下调整分辨率缩放，今天同事笔记本缩放调到175%，直接把我页面整废了，抓紧修一下！

<!--more-->

正常看着是没什么毛病的

![image-20230428171907202](解决echarts在高分屏下高度异常问题\image-20230428171907202.png)

然而调一下系统缩放，到125%

![image-20230428171950804](解决echarts在高分屏下高度异常问题\image-20230428171950804.png)

其中图表的height算法是这样的

```javascript
const clientHeight= (window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight)
const lineChartHeight = clientHeight - 475
```

lineChart:height = 浏览器高度 - header高度

然而实际缩放的时候，header高度并不是写死的475px，所以我的做法是

```javascript
const clientHeight= (window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight)
const lineChartHeight = clientHeight - 475 / ratio
```

