---
title: 解决focus时获取不到selectionStart
tags:
  - web
abbrlink: d11c887d
date: 2022-09-14 07:54:55
---

最近写浮墨笔记要同步光标位置，奈何focus时获取selectionStart总为0，找到两种解决办法。

<!--more-->

## 问题

需求是这样：

点击别处（非输入框）后隐藏联想框，点击输入框后显示联想框，这时候需要获取光标位置以便于过滤用户输入的标签。

思路是捕获focus事件，在focus事件里拿到selectionStart, selectionEnd同步位置。

这里我从网上找的demo演示一下：[Angular (forked) - StackBlitz](https://stackblitz.com/edit/angular-ifvkuz?file=src%2Fapp%2Fapp.component.ts)

![QQ20220914-080216-HD](解决focus时获取不到selectionStart/QQ20220914-080216-HD.gif)

代码是正常地focus事件：

```javascript
import { Component } from '@angular/core';

@Component({
  selector: 'my-app',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css'],
})
export class AppComponent {
  name = 'Angular';

  selectionStart = 0;
  selectionEnd = 0;

  onFocus(event) {
    let selectionIndex = (this.selectionStart = event.target.selectionStart);
    this.selectionEnd = event.target.selectionEnd;
    console.log(selectionIndex); // 0
  }
}
```

## 解决

一种方法是，把获取坐标写在`setTimeout`里面。

```javascript
onFocus(event) {
	setTimeout(()=>{
    let selectionIndex = (this.selectionStart = event.target.selectionStart);
    this.selectionEnd = event.target.selectionEnd;
    console.log(selectionIndex);
  }, 0)
}
```

另一种方法，不捕获`focus`事件，而是捕获`mouseup`事件。

两种方法都可以解决问题

[Angular (forked) - StackBlitz](https://stackblitz.com/edit/angular-wrphcs?file=src%2Fapp%2Fapp.component.html)

![QQ20220914-082935-HD](解决focus时获取不到selectionStart/QQ20220914-082935-HD.gif)
