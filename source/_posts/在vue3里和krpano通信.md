---
title: 在vue3里和krpano通信
abbrlink: 2ab2f208
date: 2023-01-14 08:23:03
tags:
- krpano
- vue
---

如何在krpano中执行vue组件里的函数，而不是放到全局导致拿不到context？

经过一天的摸索，找到了一种写法。

<!--more-->

使用原生Vanilla.js编写的函数默认放到全局，krpano里面填上函数名即可调用。

但是vue编写的函数会被编译，而且声明不在全局，所以krpano找不到vue内声明的函数。

那我直接把vue内部写的函数挂到全局就好了嘛，也就是挂到window上。

简单试一下

```vue
<script lang="ts" setup>
window.testhaha = () => {
  console.log('testhaha')
}
krpano.set(`hotspot[hotspot_name].onclick`, 'js(testhaha())')
</script>
```

这里实测可以打印`testhaha`

