---
title: 使用AI模仿孙燕姿唱歌
tags:
  - AI
abbrlink: 20c76524
date: 2023-06-30 07:59:38
---
偶然看到孙燕姿唱最近流行的歌，音色很符合她，听完才发现竟然不是本人唱的，而是AI依据原歌曲和孙女士声音模型混合的结果。
<!--more-->

## 效果预览

[【AI孙燕姿】《遥远的歌》Cover 刘惜君_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1o14y1Z7EK/?vd_source=1ec8998e58f8cdf9b1492f4de118be1a)
这位up主训练的不错，我的耳朵听不出来是AI唱的，而且还有孙燕姿的风格。

## 准备工作
- 安装[Python](https://www.python.org/downloads/)，并配置环境变量 (本人使用python3.10)
- 人声提取：Ultimate Vocal Remover - ultimatevocalremover.com/
https://github.com/Anjok07/ultimatevocalremovergui
- so-vits-svg GUI：[so-vits-svc-fork/README_zh_CN.md at main · voicepaw/so-vits-svc-fork · GitHub](https://github.com/voicepaw/so-vits-svc-fork/blob/main/README_zh_CN.md)，直接点击.bat下载
打开bat可能出现访问不了huggingface，需要给终端配置代理
```
# 编辑终端配置文件
code $PROFILE
# 将以下内容添加到文件末尾，端口需要根据你的代理软件设置，我这是clash
$env:http_proxy="http://127.0.0.1:7890"
$env:https_proxy="http://127.0.0.1:7890"
```
- 孙女士声音模型可在该博客下载：[AI天后,在线飙歌,人工智能AI孙燕姿模型应用实践，复刻《遥远的歌》，原唱晴子(Python3.10)-刘悦 (v3u.cn)](https://v3u.cn/a_id_310)，下载后解压，得到`config.json`和`G_27200.pth`

## 开始制作
1. 首先下载你想让AI唱的歌，然后使用人声提取软件ltimate Vocal Remover提取人声
![image-20230630082155166](使用AI模仿孙燕姿唱歌\image-20230630082155166.png)
如图，选择输入音频和输出位置，使用默认模型，点击Start，等待提取完成。

2. 双击bat文件打开so-vits-svg GUI
![image-20230630082028007](使用AI模仿孙燕姿唱歌\image-20230630082028007.png)
选择模型和配置文件，选择刚才提取的人声，点击infer，等待生成完毕。

推理结束后就可以在输出目录那里看到孙女士唱的歌了。
