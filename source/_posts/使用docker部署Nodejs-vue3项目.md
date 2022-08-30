---
title: 使用docker部署Nodejs+vue3项目
tags:
  - docker
abbrlink: 9300d1ee
date: 2022-08-30 07:52:50
---

最近忙完了在线考试系统的程序编写+论文，准备部署上线，为了方便选用docker，为了更方便，把mysql, redis也统统装到docker，哈哈哈哈

<!--more-->

## 编写dockerfile

首先你需要安装`docker`以及`docker-compose`。

在项目根目录新建`docker-compose.yml`，直接上我写的

```yaml
version: '3.0'
volumes:
  # 自定义共享卷，方便为nginx提供前端编译文件
  app-volume: {}
services:
  redis:
    container_name: redis
    image: daocloud.io/library/redis
    restart: on-failure
    deploy:
      resources:
        # 内存及cpu限制
        limits:
          cpus: '0.50'
          memory: 500M
    # 使用项目提供的redis配置文件
    volumes:
      - ./src/config/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  mysql:
    container_name: mysql
    image: daocloud.io/library/mysql:5.7
    restart: on-failure
    environment:
      - MYSQL_ROOT_PASSWORD=123456 # root用户密码
    volumes:
      # 第一次构建执行的sql脚本
      - ./src/config/sql:/docker-entrypoint-initdb.d/
      # mysql配置文件
      - ./src/config/my.cnf:/etc/my.cnf
  nginx:
    container_name: nginx
    image: daocloud.io/library/nginx:latest
    restart: on-failure
    ports:
      - 3000:80
    volumes:
      # nginx配置文件
      - ./vhosts.conf:/etc/nginx/conf.d/default.conf
      # 共享前端打包目录
      - app-volume:/usr/share/nginx/exam

  server:
    # 用来指定一个包含Dockerfile文件的路径。一般是当前目录. 将build并生成一个随机命名的镜像
    build: .
    # 镜像
    image: nest-server:v0.1
    # 容器名称
    container_name: nest-server
    # 指定与部署和运行服务相关的配置（restart: always关机或者重启docker同时重启容器）
    restart: always
    depends_on:
      - mysql
      - redis
      - website
    volumes:
      # 共享前端打包目录
      - app-volume:/frontend-build/dist
```

上面用到的文件，分别展示`mysql`, `nginx`, `dockerfile`

### mysql

```sql
# src/config/sql/init.sql
CREATE DATABASE  `exam` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

```bash
# Default Homebrew MySQL server config
[mysqld]
# Only allow connections from localhost
bind-address = 0.0.0.0
datadir = /usr/local/var/mysql
sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
```

### nginx

```nginx
# vhost.conf
server {
    listen       80;
    server_name  localhost;
    access_log  /var/log/nginx/host.access.log  main;
    error_log  /var/log/nginx/error.log  error;
    
    # gzip config
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 9;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;
    gzip_disable "MSIE [1-6]\.";

  	# 将前缀带api的请求转发到nestjs服务器上（服务器路由不带api）
    location ^~/api/ {
        proxy_pass http://server:3000/;
        proxy_redirect default;
    }
  	# 将所有路径转发到前端项目上
    location / {
        root   /usr/share/nginx/exam;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/exam;
    }
    
}
```

### dockerfile

```dockerfile
# dockerfile
FROM node:lts-alpine
# 装个git
RUN sed -i "s/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g" /etc/apk/repositories 
RUN apk update && \
    apk add --update git && \
    apk add --update openssh
# 设置淘宝源
RUN yarn config set registry https://registry.npm.taobao.org/
# 打包前端项目
WORKDIR /frontend-build
RUN git clone https://github.com/sunzehui/vue3-examination.git ./
RUN yarn
# 防止因内存不足导致打包失败
RUN node --max_old_space_size=2000 ./node_modules/vite/bin/vite.js build
RUN rm -rf ./node_modules
# 打包后端项目
WORKDIR /backend-build
# 将本项目拷贝到容器内
COPY . .
RUN yarn install --production
RUN yarn build
# 运行时执行
CMD yarn start:prod
```

## 在服务器上部署

服务器是`Ubuntu`，安装上`docker`以及`docker-compose`。

进入项目根目录，直接一行命令

```bash
docker-compose up
```

第一次运行要先编译出来镜像

![image-20220830082907519](使用docker部署Nodejs-vue3项目/image-20220830082907519.png)

这是其他依赖已经构建好的镜像，需要下载

![image-20220830083020008](使用docker部署Nodejs-vue3项目/image-20220830083020008.png)

然后就启动成功了

![image-20220830083226144](使用docker部署Nodejs-vue3项目/image-20220830083226144.png)



## 排错经历

### 修改代码文件重新启动无效

重新启动是走的已经编译好的镜像，如果要想更改生效，需要重新编译镜像

```bash
docker-compose build --no-cache
```

### 修改nginx配置文件重新编译启动无效

这个配置文件是宿主机与容器之间共享的，需要将共享卷删掉再编译

```bash
docker-compose down -v
```

为了方便吧，总之一句话

```bash
alias dccbr='docker-compose down -v && docker-compose build --no-cache && docker-compose up'
```

### 前端打包失败，原因是未安装vite

构建前端的时候不能`yarn install --production`,而应该全部安装，包括`devDependencies`

### 后端数据库sql_mode问题

自带sql_mode中有一条：`only_full_group_by`，要删掉，需要永久更改sql_mode，所以指定配置文件

### 多个容器间互相通信

因为两个容器是互相隔离的。

本来的想法是mysql开放端口xxxx到宿主机，然后nest-server连接宿主机的ip，并且端口填mysql共享给宿主机的端口。

后来了解到docker有一套机制，host直接填compose中的service名就可以了



## 闲扯

删除镜像太麻烦了，今天更新宝塔面板的时候，发现支持docker可视化了，能全选删除，还挺方便的。

![image-20220830092347802](使用docker部署Nodejs-vue3项目/image-20220830092347802.png)

不过上面写了企业版专享功能，以后就收费了吧，装了个其他的，界面还不错。

![image-20220830092555310](使用docker部署Nodejs-vue3项目/image-20220830092555310.png)

安装方法：

```bash
docker pull pottava/docker-webui
docker run -p 9000:9000 --rm -v /var/run/docker.sock:/var/run/docker.sock pottava/docker-webui
```

