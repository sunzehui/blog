---
title: 开发krpano语法高亮vscode插件记录
tags:
  - vscode
abbrlink: 5e604b43
date: 2022-11-13 17:09:28
---

迫于对语法高亮的追求，致我看krpano代码时晕头转向，遂开发一款语法高亮插件帮助阅读和代码编写。

<!--more-->

效果：（前两天写的拖拽热点代码）

![test](开发krpano语法高亮vscode插件记录/test.png)

看着总算舒服了些，并且敲关键词的时候也有联想提示，不过没啥意思。至少单词敲错不会高亮能减少bug。

[sunzehui/vscode-krpano (github.com)](https://github.com/sunzehui/vscode-krpano)

这个是源码，已经发布到vscode插件市场了



语法高亮只是用正则表达式匹配的文本，为文本块分配不同的语法类型，然后染色器去按照类型染色。

不会很复杂，这些功能只是最基础的只能叫做编辑器的功能

要实现更复杂的比如代码跳转，hover提示，等等，需要开发LSP（language server protocol）服务器，目前还在研究。在看vetur代码，调试打印不出来log，很烦。