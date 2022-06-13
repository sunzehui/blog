---
title: 解决zindex失效
abbrlink: f918b5d4
date: 2022-04-24 18:20:31
tags:
- coding
---

## 问题演示

![CPT2204241753-501x128](解决zindex失效/CPT2204241753-501x128-16507940777251.gif)

本来鼠标滑过应该是有高亮来着，详细看上篇文章：

[实现滑动动画菜单 | 孙泽辉 (hui.zone)](./7e11ffc.html)

TLNR: [解决方案](#解决方案)

## 解决过程

怀疑是`highlight`的z轴太小了，改一下

![image-20220424175926240](解决zindex失效/image-20220424175926240.png)

文字又被盖住了，再把文字z轴改一下

![image-20220424180021786](解决zindex失效/image-20220424180021786.png)

修改文字没反应了。。

然后一通乱改，改到怀疑人生

谷歌一下发现是因为header用了`position: fixed`，会触发创建一个新的层叠上下文，它的父层叠上下文变成了header，所以ul只能在header的内部进行层叠比较。这也就是大家熟听的“从父原则”。



## 解决方案

给父级元素设置z轴层级是0就可以了，这样层级之间就可以有根据进行比较了！

![image-20220424181745154](解决zindex失效/image-20220424181745154.png)



参考资料：

[CSS Z-Index Not Working? How to Fix It Using Stack Order (freecodecamp.org)](https://www.freecodecamp.org/news/css-z-index-not-working/)

[position为fixed时设置z-index失效 - Typecho主题模板站 (lpmcn.com)](https://www.lpmcn.com/skill/336.html)

[position为fixed时设置z-index失效 | 李大公子 (lifuzhen.github.io)](