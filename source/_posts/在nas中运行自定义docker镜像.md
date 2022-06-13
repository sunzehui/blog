---
title: 在nas中运行自定义docker镜像
date: 2022-04-11 7:04:46
abbrlink: abcd123
tags:
- docker
- nas
---


docker容易上手且不限制环境，是部署项目的好工具！

<!--more-->

## 起因

![门面](在nas中运行自定义docker镜像/Screenshot_20220411_055754_com.taobao.idlefish.jpg)

最近在闲鱼上提供配置开发环境服务，一位兄弟的需求是在电脑上配置好脚本的环境，最初是想要把`nodejs`脚本打包成`.exe`（window下可执行文件），这就触及到我的知识盲区了，还真没搞过。最后选择的方案是使用`docker`创建镜像扔到他的`nas`上面。

![和买家交流](在nas中运行自定义docker镜像/Screenshot_20220411_055915_com.taobao.idlefish.jpg)

（100不是我要的，大哥给的，谢谢我大哥！）





## 编写dockerfile

一共是两个项目，都差不多

流程是：先从本地创建(build)出镜像导出，然后扔到`nas`上导入

[HFrost0/Lighting-bilibili-download: 快如闪电的bilibili下载工具，基于Python现代Async特性，高速批量下载整部动漫，电视剧，up投稿等 (github.com)](https://github.com/HFrost0/Lighting-bilibili-download)

这个项目的依赖软件有`Python`、`FFmepg`，dockerfile如下

```dockerfile
# 基于python3.8镜像创建
FROM python:3.8
# 复制运行目录下的文件到容器
WORKDIR /test
ADD . /test

# 跳过不必要的二进制文件（好像是）
ENV DEBIAN_FRONTEND=noninteractive

# 换源，装FFmpeg
RUN sed -i s@/deb.debian.org/@/mirrors.aliyun.com/@g /etc/apt/sources.list
RUN apt-get clean
RUN apt-get update && apt-get install -y ffmpeg
# 安装python依赖
RUN pip install 'httpx[http2]' rich json5
```

没什么复杂命令，直接创建镜像了

```shell
# 创建镜像
docker build -t bili-down -f dockerfile .
# 运行容器
docker run -it bili-down
# 查看所有容器
docker container ls -a
# 找到bili-down填写容器id，导出镜像bili-down.tar
docker export xxxxxxx > bili-down.tar
```



## 配置nas

镜像导出之后上传到同学的`nas`上面，导入容器

![image-20220411062110049](在nas中运行自定义docker镜像/image-20220411062110049.png)

（已经晕了，导入容器实际上导入到映像里了，现在从映象创建容器）



正常配置的话启动下载完毕就自动关闭了，我想让他保持运行，并且后续可以进入终端执行命令，运行进入默认bash。。

![image-20220411070257300](在nas中运行自定义docker镜像/image-20220411070257300.png)

然后和宿主机共享文件夹，将下载的文件和宿主机互通。

![image-20220411070122476](在nas中运行自定义docker镜像/image-20220411070122476.png)

现在通过容器的终端运行命令，正常下载了！任务结束。

![image-20220411070500824](在nas中运行自定义docker镜像/image-20220411070500824.png)



![image-20220411070409548](在nas中运行自定义docker镜像/image-20220411070409548.png)



## 总结

`nas`这个没搞过，其实没多少信心，好在同学积极配合，让我能摸一把nas。

这种被赶着走的感觉为我松散的生活上了上弦，学习效率更高，更投入。

我不敢说配置环境完全学不到什么，公司里貌似也有nas，某天偶尔被老板问到：有没有搞过nas？我便可从容地回答。下一个被pass的，将不会是我。

