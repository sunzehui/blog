---
title: 阅读data.task并实现简易版Task
tags:
  - coding
abbrlink: 7fd036cc
date: 2022-07-04 13:16:49
---

今天做题遇到题目用到了`data.task`这个包，里面的`Task`实现很巧妙，类似`Promise`但又不是，简单对比分析一下。

<!--more-->

## Task 使用方法

当时看这个代码看入迷了

```javascript
// 练习 3
// ==========
// 使用 getPost() 然后以 post 的 id 调用 getComments()
var getPost = function (i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res({ id: i, title: "Love them tasks" });
    }, 300);
  });
};

var getComments = function (i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res([
        { post_id: i, body: "This book should be illegal" },
        { post_id: i, body: "Monads are like smelly shallots" },
      ]);
    }, 300);
  });
};
// getPost 返回task 使用chain取出（至于为什么可以取出，看下面源码）
const getCommentsById = _.compose(getComments, _.prop("id"));
var ex3 = _.compose(_.chain(getCommentsById), getPost);

// 有必要写一下测试代码
it("Exercise 3", function (done) {
    E.ex3(13).fork(console.log, function (res) {
        assert.deepEqual(res.map(_.prop("post_id")), [13, 13]);
        done();
    });
});
```

这个是可以跑得，并且通过了测试，相信你的眼睛。

好奇`getPost`一个异步任务怎么可能将两个函数衔接的如此流畅并且一点副作用没有！

其实这类似`RXjs`的网络流，处理函数都在后面等着数据流。



## Task 源码

于是点开`Task`的源码

```javascript
module.exports = Task;

function Task(computation, cleanup) {
    // 保存传入的callback
    this.fork = computation;
    
    this.cleanup = cleanup || function() {};
}
// 类似 Promise.resolve
Task.prototype.of = function _of(b) {
    return new Task(function(_, resolve) {
        return resolve(b);
    });
};
Task.prototype.rejected = function _rejected(a) {
    return new Task(function(reject) {
        return reject(a);
    });
};

// -- Functor ----------------------------------------------------------

/**
 * Transforms the successful value of the `Task[α, β]` using a regular unary
 * function.
 *
 * @summary @Task[α, β] => (β → γ) → Task[α, γ]
 */
// 对数据操作
Task.prototype.map = function _map(f) {
    var fork = this.fork;
    var cleanup = this.cleanup;

    return new Task(function(reject, resolve) {
        return fork(function(a) {
            return reject(a);
        }, function(b) {
            // 实质上是在resolve之前，类似adapter
            return resolve(f(b));
        });
    }, cleanup);
};

// -- Chain ------------------------------------------------------------

/**
 * Transforms the succesful value of the `Task[α, β]` using a function to a
 * monad.
 *
 * @summary @Task[α, β] => (β → Task[α, γ]) → Task[α, γ]
 */
// 衔接 Task
Task.prototype.chain = function _chain(f) {
    var fork = this.fork;
    var cleanup = this.cleanup;
	// 返回新的 Task 类似 Promise.then
    return new Task(function(reject, resolve) {
        // 执行当前fork后
        // 在success回调继续fork
        return fork(function(a) {
            return reject(a);
        }, function(b) {
            // 对传入的fork（被衔接）执行fork
            // 将resolve权移交新的fork
            return f(b).fork(reject, resolve);
        });
    }, cleanup);
};
```



## 手动实现

思路分析：

刚才看源码了，把传入的函数保存起来，推迟到实例化对象调用fork之后执行回调，然后resolve一连串都跑起来了，太秀了

实现如下：

```javascript
// sunzehui 的 Task ，简称 STask
class STask {
    fork = () => {};
    constructor(cb) {
        this.fork = cb;
    }
    //   f 是个返回STask的函数
    concat(f) {
        const thisTask = this.fork;
        return new STask(function (res, rej) {
            thisTask(
                function (r) {
                    f(r).fork(res, rej);
                },
                function (e) {
                    rej(e);
                }
            );
        });
    }
}
```

不敢相信，竟然几行代码写出来的`Task`看看能不能用？

```javascript
// 计时5秒
let i = 5;
setInterval(() => console.log(i--), 1000);

const find = () =>
  new STask((resolve, reject) => {
    setTimeout(() => {
        resolve("函数式编程指北");
    }, 3000);
});
const read = (book) =>
  new STask((resolve, reject) => {
    setTimeout(() => {
        resolve("I am reading " + book);
    }, 3000);
});
// 拼接任务
const readBook = find().concat(read);
readBook.fork(function (res) {
    console.log("success：", res);
});

/**
~/part2_exercises on master!  12:29:37
$ node test.js
5
4
3
2
1
success： 函数式编程指北
0
**/
```

## 总结

简单实现了两个任务拼接而已。。

上面说到，`fork`时会将在构造函数传入的`callback`执行，但`Promise`不是，它会在创建时就执行，与其对应的`then`仅仅是注册成功/失败回调

`Promise` 是 `pubsub` 模式，而 `data.task` 这个是利用`continuation-passing-style`

[By example: Continuation-passing style in JavaScript (might.net)](https://matt.might.net/articles/by-example-continuation-passing-style/)

