---
title: 解决在ios上以视频作为背景弹出原生播放器问题
date: 2023-05-08 13:55:44
tags:
- coding
---

RT

<!--more-->



```javascript
const videoEl = document.querySelector("#video");
const ua = navigator.userAgent.toLocaleLowerCase();
// x5内核
if (ua.match(/tencenttraveler/) != null || ua.match(/qqbrowse/) != null) {
    videoEl.setAttribute("x-webkit-airplay", true);
    videoEl.setAttribute("x5-playsinline", true);
    videoEl.setAttribute("webkit-playsinline", true);
    videoEl.setAttribute("playsinline", true);
} else {
    // other
    videoEl.setAttribute("webkit-playsinline", "true");
    videoEl.setAttribute("playsinline", "true");
}
```

