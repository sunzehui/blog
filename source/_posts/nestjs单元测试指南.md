---
title: nestjs单元测试指南
tags:
  - UT
abbrlink: 7db59b4f
date: 2022-07-12 10:24:36
---

刚刚接触测试，发现坑太多了，有必要记录一下。

<!--more-->

测试效果：

![image-20220712102605463](nestjs单元测试指南/image-20220712102605463.png)

## 配置测试环境

我的项目是使用`@nest/cli`创建的，它只有一个`app.controller.spec.ts`，但是引入我写的module就会报很多错

## 路径别名

首先解决`tsconfig.ts`的路径别名问题，如果你在`app.module.ts`中写了路径别名，那么`jest`处理不了

```javascript
import { UserModule } from '@/user/user.module';
// 报错
/*
 FAIL  src/user/test/user.controller.spec.ts
  ● Test suite failed to run

    Cannot find module '@/user/entities/user.entity' from 'user/test/user.controller.spec.ts'

      1 | import { Test, TestingModule } from '@nestjs/testing';
      2 | import { TypeOrmModule } from '@nestjs/typeorm';
    > 3 | import { User } from '@/user/entities/user.entity';
        | ^
      4 | import { UserController } from '../user.controller';
      5 | import { UserService } from '../user.service';
      6 | import connectionCfg from '@/config/database';

      at Resolver._throwModNotFoundError (../node_modules/.pnpm/jest-resolve@28.1.1/node_modules/jest-resolve/build/resolver.js:491:11)
      at Object.<anonymous> (user/test/user.controller.spec.ts:3:1)

Test Suites: 1 failed, 1 passed, 2 total */
```

解决方法：在`package.json`的`jest`配置中，添加

```json
// package.json
{
  "name": "examination",
  "version": "0.0.1",
  "description": "",
  "author": "",
  "dependencies": {
  },
  "devDependencies": {
  },
  // !!!!!!!!!!!!!!!!!!!!!!!!!!!
  "jest": {
    "rootDir": "src",
    "moduleNameMapper": {
      "@/(.*)$": "<rootDir>/$1"
    },
    // ...
  }
  // !!!!!!!!!!!!!!!!!!!!!!!!!!!
}
```



## 配置typeorm

typeorm的配置也要一并添加进test中

这是我的`user.controller.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from '@/user/entities/user.entity';
import { UserController } from '../user.controller';
import { UserService } from '../user.service';
import connectionCfg from '@/config/database';

describe('AppController', () => {
  let userController: UserController;

  beforeEach(async () => {
    // 这里注入 User,和配置文件
    const user: TestingModule = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({ ...connectionCfg, entities: [User] }),
        TypeOrmModule.forFeature([User]),
      ],
      controllers: [UserController],
      providers: [UserService],
      exports: [UserService],
    }).compile();

    userController = user.get<UserController>(UserController);
  });

  describe('register', () => {
    it('should register!"', async () => {
      const res = await userController.register({
        username: 'sunzehui',
        password: 'sunzehui',
      });
      console.log(res);
    });
  });
});
```

注意配置文件要把原有的`entities`替换成自己手动导入的，因为之前是从`dist`拿的，现在拿不到。
