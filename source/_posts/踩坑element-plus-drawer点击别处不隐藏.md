---
title: 踩坑element-plus-drawer点击别处不隐藏
tags:
  - vue
  - coding
abbrlink: d08010b3
date: 2022-09-04 17:36:08
---

明明示例可以点击别处隐藏，我的配置和他一样，原因竟是没有引入遮罩样式！

<!--more-->

![CPT2209041739-974x596](踩坑element-plus-drawer点击别处不隐藏/CPT2209041739-974x596.gif)

上面是bug效果，点drawer以外的东西没反应，而且狂点也没反应！



认真地和示例程序对比后发现，人家的有一层遮罩，而我的没有。

![CPT2209041743-1053x792](踩坑element-plus-drawer点击别处不隐藏/CPT2209041743-1053x792.gif)

此时立刻想到可能是没引入遮罩样式，引入后就解决了。

在`main.ts`中引入遮罩样式

```typescript
import { createApp } from "vue";
import "element-plus/theme-chalk/el-overlay.css"; // 遮罩层样式
import App from "./App.vue";

const app = createApp(App);
app.mount("#app");
```

