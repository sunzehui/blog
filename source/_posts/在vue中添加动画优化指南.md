---
title: 在vue中添加动画优化指南
tags:
  - vue
abbrlink: df5ab725
date: 2022-11-27 09:41:33
---

最近在页面间跳转加了点动画，发现有卡顿、闪屏现象，经过一顿调试后，问题解决（几乎），谈一下常见优化思路以及我的操作。

<!--more-->

## 问题复现

先看之前动画有问题的样子

![动画](在vue中添加动画优化指南/动画.gif)
明显上面有些闪动，体验很差。

## 问题解决

动画一闪而过不方便调试，翻阅 vue 文档发现，transition 组件可以传入动画时长，以便在动画后将 dom 清除。
这里给 transition 组件传入 duration 属性，时长设置特别大，方便调试

```html
<Transition :name="transitionName" :duration="10000000" mode="out-in">
  <KeepAlive :max="10">
    <RouterView v-slot="{ Component, route }">
      <Component :is="Component" :key="route.path" />
    </RouterView>
  </KeepAlive>
</Transition>
```

这是动画结束后不清除动画后的 dom 的样子

![动画](在vue中添加动画优化指南/动画-1669514986908.gif)

‘志愿服务’这个页面被挤扁了，或者说宽度没有占满

除了这个页面，别的都正常。

最终发现是宽度没有设置的问题，给页面宽度写 100%就可以了。
另外，红色 Title 有抖动问题，之前 Title 是用 fixed 定位写的，换成 sticky 好多了。

关于动画，

```css
.slide-right-enter-active,
.slide-right-leave-active,
.slide-left-enter-active,
.slide-left-leave-active {
  // 性能优化
  will-change: transform;
  transition: all 1000ms;
  // 绝对定位脱离文档流动画效果更好
  position: absolute;
  // 切换时隐藏滚动条
  .page .view-box {
    overflow: hidden;
  }
}

.slide-right-enter {
  opacity: 0;
  transform: translate3d(-100%, 0, 0);
}

.slide-right-leave-active {
  opacity: 0;
  transform: translate3d(100%, 0, 0);
}

.slide-left-enter {
  opacity: 0;
  transform: translate3d(100%, 0, 0);
}

.slide-left-leave-active {
  opacity: 0;
  transform: translate3d(-100%, 0, 0);
}
```

效果



![动画](在vue中添加动画优化指南/动画-1669515780840.gif)



但是仍有不足，如果滚动条特别粗，会有抖动的现象，不过手机端看不出来，不修了。

![动画](在vue中添加动画优化指南/动画-1669516168822.gif)

我能想到的方案有三种

一个是给页面加滚动条的padding

另一个是自定义滚动条，absolute不参与文档流。

或者使用overflow: overlay 忽略滚动条，参考：[修复一个因为 scrollbar 占据空间导致的 bug - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1640410)