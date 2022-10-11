---
title: 在vue2中使用setup和compositionAPI
tags:
  - vue
abbrlink: 33583c40
date: 2022-10-12 06:51:14
---

最近写项目用到了vue2，以为要开始痛苦的optionAPI编程，没想到vue2竟然支持compositionAPI了，甚至能在setup里写代码！

<!--more-->

下面是vue2的setup写法，用起来和vue3差不多了，还支持pinia！



刚上手我还不敢相信这是用的vue2，赶紧翻看`package.json`，哦，vue2.7



使用一天下来（没写多少逻辑，光糊页面了），发现**唯一不同的就是：获取当前页面路由信息**

vue3里可以用

```javascript
// vue-router ^4.0
import { useRoute } from 'vue-router';
const route = useRoute();
route.path // /home
route.query // { foo: 'bar' }
```

但是`vue-router`版本必须是4以上，我这里的版本号是3，所以没有`useRoute`这个函数。

那vue2中的获取当前route怎么写？

```js
export default {
    // ...
    mounted(){
        const route = this.$route;
        route.path // /home
    }
}
```

要从this中获取，那我setup里this是null了，走进死胡同了。

最终找到一个vue2-router的包：[ambit-tsai/vue2-helpers: 🔧 A util package to use Vue 2 with Composition API easily (github.com)](https://github.com/ambit-tsai/vue2-helpers#vue2-helpers)

用法大概是：

```javascript
import { useRoute } from 'vue2-helpers/vue-router';

const route = useRoute();
```

这么厉害？看一下源码，关键源码：

```typescript
// https://github.com/ambit-tsai/vue2-helpers/blob/for-vue-2.7/src/vue-router.ts#L52
export function useRouter(): Router {
    // 获取当前实例
    const inst = getCurrentInstance();
    if (inst) {
        // 从当前实例里拿$router
        return inst.proxy.$root.$router as Router;
    }
    warn(OUT_OF_SCOPE);
    return undefined as any;
}

let currentRoute: Route;

export function useRoute(): RouteLocationNormalizedLoaded {
    const router = useRouter();
    if (!router) {
        return undefined as any;
    }
    if (!currentRoute) {
        const scope = effectScope(true);
        scope.run(() => {
            // router.currentRoute是当前route
            currentRoute = reactive(assign({}, router.currentRoute)) as any;
            router.afterEach((to) => {
                assign(currentRoute, to);
            });
        });
    }
    return currentRoute;
}
```

从上面代码可以看出，关键在于`getCurrentInstance()`，其实从这里就可以拿到this上的东西了，可以直接拿`$route`

```javascript
// 获取当前实例
const inst = getCurrentInstance();
if (inst) {
    // inst.proxy.$root === this
    return inst.proxy.$root.$route;
}
```

最终我没有引入这个包，直接这么写的