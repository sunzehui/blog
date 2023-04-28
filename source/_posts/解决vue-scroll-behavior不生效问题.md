---
title: 解决vue-scroll-behavior不生效问题
date: 2023-04-28 16:44:54
tags:
- coding
---

之前研究过vue-scroll-behavior源码，操作也都清楚，奈何配置好了，切换页面后无效，也不滚动到指定位置。。

<!--more-->

## 页面嵌套问题

正常来说，想要实现返回列表后保留原来位置，需要给列表页面添加keepAlive，保证数据切换页面不会丢失。

因为我在App.vue里面加了

```vue
// App.vue
<transition :name="$route.meta.transition" duration="300">
    <template v-if="noKeepAliveRoute">
		<router-view v-slot="{ Component, route }">
    		<component :is="Component" :key="route.path" />
        </router-view>
    </template>
    <template v-else>
        <KeepAlive :max="10">
            <router-view v-slot="{ Component, route }">
            	<component :is="Component" :key="route.path" />
            </router-view>
        </KeepAlive>
    </template>
</transition>
```

并且我确定我的列表页不在noKeepAliveRoute之列，所以是能够缓存列表然后滚动的。

实际却是切换回来修改scrollTop不管改什么值都是0。

随后我在生命周期created中打印个字符串'inited'，切换页面后发现它打印了两遍：第一次一遍，再切页面一遍。

![动画](解决vue-scroll-behavior不生效问题\动画.gif)

说明该页面没有缓存上。



遂检查路由，发现这个页面嵌了不止一层，所以说，在最近的一层上面加上keep-alive就好了。

## 动画问题

解决了吗，仍然没有，虽然上面的可以了，但是有些页面还是如以前一样。

我的方法是把源码copy到本地，引入后打断点。

果然，在设置返回列表页面的scrollTop的代码断点中，我发现列表页还没加载出来，还停留在详情页时就执行了。

![image-20230428170149228](解决vue-scroll-behavior不生效问题\image-20230428170149228.png)

碍于本人对加载时机极其敏感（写原生写的），这时候列表页面还没加载出来，设置scrollTop一定是无效的，所以应该等页面加载出来再设置scrollTop。

尝试setTimeout发现需要指定一个猜测出来的毫秒数，不过有些页面加载速度不一样，不能用这种土方法。

比较和之前修复的页面的区别，发现该页面添加了页面切换动画，记得动画是先生成了下一组件dom后添加css动画的

![image-20230428170749677](解决vue-scroll-behavior不生效问题\image-20230428170749677.png)

果然，现在#app上有两个页面，vueScrollBehavior插件用的querySelector选中元素默认选中了第一个，而第一个是旧的页面，设置旧的页面是没有意义的，我们要设置新的页面。

为了省事，干脆把querySeletor改成querySeletorAll即可。

```javascript
vue-scroll-behavior\src\utils\helpers.js
/**
 * Setting Scroll Position
 */
export function setScrollPosition(Vue, position = { x: 0, y: 0 }) {
  const scrollTo = () => {
    Vue.nextTick(() => {
+      const el = document.querySelectorAll(vueScrollBehavior._el)
+      if (el.length) {
+        el.forEach((el) => {
+          el.scrollLeft = position.x
+          el.scrollTop = position.y
+          el.scrollTo(position.x, position.y)
+        })
      }
      else {
        window.scrollTo(position.x, position.y)
      }
    })
  }
  if (vueScrollBehavior._delay > 0)
    setTimeout(scrollTo, vueScrollBehavior._delay)

  else
    scrollTo()
}
```

问题成功解决了！

![动画](解决vue-scroll-behavior不生效问题\动画-1682673109569.gif)