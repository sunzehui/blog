---
title: 使用jest单元测试
tags:
  - UT
abbrlink: f5769b3b
mathjax: true
date: 2022-05-30 11:38:06
---
今天写代码碰到一处逻辑判断，突然想起来使用单元测试把所有可能的结果跑一遍，全方位打击，那一定不会再出bug了吧！

<!--more-->

## 逻辑复现

我的逻辑很简单，是前端路由守卫，判断用户有没有登录，且这个路由需不需要登录才能访问，代码如下：

```typescript
import { ElMessage } from "element-plus";
import { useUserStore } from "@/store/user";
import { Router } from "vue-router";

export default function loginGuard(router: Router) {
  router.beforeEach((to, from, next) => {
    const thisRoutePublic = to.meta.publicRoute;

    const userStore = useUserStore();
    const isAuthenticated = userStore.isAuthenticated;

    if (!isAuthenticated && !thisRoutePublic) {
      ElMessage.error("请先登录");
      next({ name: "Login" });
    } else next();
  });
}
```

如果账号没登录，而且路由需要认证的话，跳转到登录页。

看这代码这么多叹号很烦，想着能不能像是数学乘法分配律那样，提个感叹号出来，幻想着是这样：

```typescript
if (!(isAuthenticated && thisRoutePublic)) {
    // do something...
};
```

刚想用脑子模拟一下算算，就感到头晕恶心马上要呕吐了，我立即停止联想，拿出`jest`编写测试用例：

```typescript
// calc.ts
class Calc {
  // 需要登录是false，不需要登录通过验证是true
  router(isAuthenticated: boolean, thisRoutePublic: boolean) {
    if (!(thisRoutePublic || isAuthenticated)) {
      return false;
    } else {
      return true;
    }
  }
}

export default new Calc();
```

这个是我把逻辑抽出来，单独用来测试的。

```typescript
// calc.spec.ts
import calc from './calc';
// 一些方便观察的枚举
const unLogin = false;
const isLogin = true;
const isPublic = true;
const isPrivate = false;
const pass = true;
const reject = false;
describe('Calc', () => {
  // 需要登录是false，不需要登录通过验证是true
  test('没有登录，当前页面不需要认证', () => {
    const guardResult = calc.router(unLogin, isPublic);
    expect(guardResult).toBe(pass);
  });

  test('没有登录，当前页面需要认证', () => {
    const guardResult = calc.router(unLogin, isPrivate);
    expect(guardResult).toBe(reject);
  });

  test('有登录，当前页面不需要认证', () => {
    const guardResult = calc.router(isLogin, isPublic);
    expect(guardResult).toBe(pass);
  });

  test('有登录，当前页面需要认证', () => {
    const guardResult = calc.router(isLogin, isPrivate);
    expect(guardResult).toBe(pass);
  });
});
```

实际上也就两个变量排列组合，四个测试的样子。。。

## 跑测试用例

正常用之前的两个取反，是可以跑通的：

![image-20220530110806132](使用jest单元测试/image-20220530110806132.png)

然后，按照我的想法，提个取反出来：

![image-20220530110853300](使用jest单元测试/image-20220530110853300.png)

哈哈哈，算的真快，直接清晰的展示出来了结果

好了，测试不通过，证明我的想法是错的，这时才想起来这不就是逻辑论里的德摩根定律吗，套公式就好了：

$$¬(A∨B)⇔[(¬A)∧(¬B)]$$

$$¬(A∧B)⇔[(¬A)∨(¬B)]$$

所以我是非A合取非B，用第一个，中间应该是析取，对应着逻辑或，改过来就好了

![image-20220530112857682](使用jest单元测试/image-20220530112857682.png)



## 总结

我觉得之前应该好好对待测试，并且我这个函数应该单独拎出来，而不必像今天这样手动拆，一些函数应该尽量逻辑纯粹，容易测试，也不必像之前混带着路由操作，不方便单元测试。

另外，我用的`jest`模板附在这里了，克隆下来就能跑

[hoangsetup/ts-jest-tdd-starter (github.com)](https://github.com/hoangsetup/ts-jest-tdd-starter)