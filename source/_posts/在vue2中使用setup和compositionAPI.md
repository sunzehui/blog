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



vue2可以使用`ref` 和`reactive`，没有使用proxy实现，还是用了defineProperty的getter,setter，可以放心使用！



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



10月14号补充：

试着在reactive中写数组，发现是不支持的

```javascript
const list = reactive([]);
```

报错描述的很清晰，说是defineProperty的局限性：不能监测数组内部变化。

> Composition API 使用 Vue 2 的基于 `getter/setter` 的响应式系统进行反向移植，以确保浏览器兼容性。 这意味着与 Vue 3 的基于 `proxy` 的系统存在一些重要的行为差异：
>
> ——摘自掘金

算是与vue3不同的地方，解决办法，我换`ref`用起来是可以的

```javascript
const list = ref([])
```



其他细节请看：[Vue 2.7 正式发布，代号为 Naruto - 掘金 (juejin.cn)](https://juejin.cn/post/7115361618774622216)



vue-router3已经支持use-router了！

[vue-router/composables.js at dev · vuejs/vue-router (github.com)](https://github.com/vuejs/vue-router/blob/dev/composables.js#L30)

只需：

```javascript
import { useRoute } from 'vue-router/composables'
const route = useRoute();
```



关于vue2.7中vue-router的讨论：[[功能请求\]添加对 Vue 2.7 的支持 ·问题 #3760 ·vuejs/vue-router (github.com)](https://github.com/vuejs/vue-router/issues/3760)