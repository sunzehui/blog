---
title: 使用transition组件实现b站页面切换效果
date: 2022-11-27 12:18:39
tags:
- vue
---

想为页面添加点切换效果，参考哔哩哔哩手机端APP，效果还行。

<!--more-->

## 效果展示

先看一下b站的效果

<video src="使用transition组件实现b站页面切换效果/Screenrecording_20221127_121425.mp4"></video>

点击视频卡片后，列表页稍微左移，详情页从右往左移动到视口。

返回即详情页右移出视口，展示出列表页，此时列表页从稍微左移中往右回正，平铺在视口。

实现效果：

![动画](使用transition组件实现b站页面切换效果/动画.gif)



## 实现方法

技术要点包括vue的transition组件，路由的后退和前进判断。

### transition组件

首先讲一下vue的transition组件，该组件传入name属性名后，会根据属性名绑定以下6个类名：

在进入/离开的过渡中，会有 6 个 class 切换。

1. `v-enter`：定义进入过渡的开始状态。在元素被插入之前生效，在元素被插入之后的下一帧移除。
2. `v-enter-active`：定义进入过渡生效时的状态。在整个进入过渡的阶段中应用，在元素被插入之前生效，在过渡/动画完成之后移除。这个类可以被用来定义进入过渡的过程时间，延迟和曲线函数。
3. `v-enter-to`：**2.1.8 版及以上**定义进入过渡的结束状态。在元素被插入之后下一帧生效 (与此同时 `v-enter` 被移除)，在过渡/动画完成之后移除。
4. `v-leave`：定义离开过渡的开始状态。在离开过渡被触发时立刻生效，下一帧被移除。
5. `v-leave-active`：定义离开过渡生效时的状态。在整个离开过渡的阶段中应用，在离开过渡被触发时立刻生效，在过渡/动画完成之后移除。这个类可以被用来定义离开过渡的过程时间，延迟和曲线函数。
6. `v-leave-to`：**2.1.8 版及以上**定义离开过渡的结束状态。在离开过渡被触发之后下一帧生效 (与此同时 `v-leave` 被删除)，在过渡/动画完成之后移除。

—— 摘自vue官方文档 [进入/离开 & 列表过渡 — Vue.js (vuejs.org)](https://v2.cn.vuejs.org/v2/guide/transitions.html#过渡的类名)

此时可以拆解为两类过渡，一种是点击卡片进入详情页的过渡，另一种是详情页点击返回展示列表页的过渡，下面分别叙述。





**点击卡片后展示详情页过渡：**

从列表页点击卡片后，看见展示的页面（详情页）应该是从最右边移动过来的，所以第一帧应为

```css
.slide-come-enter {
	transform: translate3d(100%, 0, 0); // 可以理解为向右偏移自身宽度
}
```

因为页面本身状态是 偏移0%，所以无需写 v-enter-to

对于列表，仅仅需要在结束帧稍稍向左移动，这里取移动自身的 20%，即：

```css
.slide-come-leave-to{
	transform: translate3d(-20%, 0, 0);
}
```

同样的无需写 v-leave，因为初始状态就是从 偏移 0% 开始。

对于两个v-enter-active, v-leave-active，该过渡类名是在动画时添加，一般用于添加过渡函数，这里使用统一的动画函数。

```css
$pageSlideDuration: 300ms;
.slide-come-leave-active,
.slide-come-enter-active {
    will-change: transform;
    transition: all $pageSlideDuration;
    position: absolute;
}
```



**在详情页点击返回按钮后，展示列表页过渡：**

从详情页点击返回按钮后，此时进入页面是列表页，对于列表页，应从原来的 -20% 偏移回正。所以初始偏移 -20%，定义第一帧应为 向左偏移 -20%。

```css
.slide-back-enter {
    transform: translate3d(-20%, 0, 0);
}
```

因为 enter-to 页面本身是 0% 所以不再写 v-enter-to

对于详情页，现在要离开，最终应向左偏移 100% 移出视口。

```css
.slide-back-leave-to{
    transform: translate3d(100%, 0, 0);
}
```

初始 v-leave 也无需写，同上。

这里要注意，leave 的详情页可能 z轴 太小，显示不出来，我这里加大了详情页 z轴，在leave-active中添加：

```css
.slide-back-leave-active{
    z-index: 999;
}
```

不知道为什么列表页进入详情页的时候，详情页 z轴正合适。。。

active时的动画函数同进入时一样，不再赘述。

现在过渡就写好了。分为 `slide-come`（进入详情页）`slide-back`（从详情页返回）

如何判断页面要加什么样的过渡呢？





### 判断前进后退以添加类名

参考网上给出的方法，为 `route` 添加优先级下标。

我这里的命名方法是

- 二级详情页：20

- 三级详情页：30

- ...

没有科学依据瞎搞的不要学。求教更科学的方式。

```javascript
// router/index.ts
// 首页路由
{
    name: 'layout-index',
    path: '/',
    component: () => import('../layouts/index.vue'),
    children: [
        {
            path: 'home',
           	meta: {
                idx: 10
            }
        },
        {
            path: 'project',
            meta: {
                idx: 11,
            },
        },
        {
            path: 'demand-form',
            meta: {
                idx: 12,
            },
        },
        // ...
    ],
},
// 详情页路由
{
    name: 'layout-default',
    path: '/',
    component: () => import('../layouts/default.vue'),
    children: [
        {
            path: '/integral',
            children: [
                {
                    path: '',
                    meta: {
                        idx: 20,
                    },
                },
                {
                    path: 'goods',
                    meta: {
                        idx: 30,
                    },
                },
            ],
        },
        {
            path: 'search',
            meta: {
                idx: 20,
            },
        },
        {
            path: 'login',
            meta: {
                idx: 20,
            },
        },
        // ...
    ],
}

```

因为我的路由是首页和详情页间切换，顶层是 App.vue，也可以说是全局切换，直接添加全局路由守卫，判断 to, from 的 meta 下标 idx 即可

```typescript
export function transition(router) {
  router.afterEach((to, from) => {
    const toIdx = (to.meta.idx)
    const fromIdx = (from.meta.idx)
    to.meta.transition = toIdx > fromIdx ? 'slide-come' : 'slide-back'
    console.log(toIdx, fromIdx,
                `isBack ${toIdx < fromIdx}`) // 10 20 isBack true
  })
}
```

以上，是本文全部内容。