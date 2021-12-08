---
title: "分阶段构建 Docker 镜像"
date: 2021-12-08T17:14:37+08:00
draft: false
tags: ["docker", "镜像"]
---



## 编写 Dockerfile

> 什么是 Dockerfile ? 
> Dockerfile 是一个用来构建镜像的文本文件, 文本内容包含了一条条构建镜像所需的指令和说明. 

### 第一阶段构建

为了使我们最后构建出来的镜像尽量的小, 我们最好是分阶段构建. 

第一个阶段我们在 go 环境下, 编译出我们的可执行二进制文件. 

构建配置如下: 

```bash
FROM golang:1.17-alpine3.13 as builder
RUN mkdir /src
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
# 安装依赖库 GCC 这些, 根据实际情况考虑是否安装
RUN apk add build-base
ADD . /src
WORKDIR /src
RUN go env -w GOPROXY=https://goproxy.cn, direct && go mod tidy
RUN GOPROXY=https://goproxy.cn go build -o app main.go  && chmod +x app
```

关键点解释: 

- From 基于那个基础镜像, 这里我们基于 go 的 1.17 版本构建, 这里最好和你本地开发环境的 go 版本保持一致. 
- RUN go env 这一行是设置 go mod 的代理, 以及根据 go.mod 文件安装依赖

其他的应该都不难理解, 有问题欢迎留言. 

### 第二阶段构建

第一阶段构建完毕, 我们就能得到一个可执行的二进制文件 app 了. 

此时我们就需要进入第二阶段构建, 真正的构建出生产用的镜像, 直接上代码: 

```bash
FROM alpine:3.12
ENV ZONEINFO=/app/zoneinfo.zip
RUN mkdir /app
WORKDIR /app

COPY --from=builder /usr/local/go/lib/time/zoneinfo.zip /app
COPY --from=builder /src/app /app
ENTRYPOINT  ["./app"]
EXPOSE 80
```

关键点解释: 

- 第一个 COPY 是复制第一阶段里面的时区, 可以根据实际情况选择是否复制. 
- 第二个 COPY 是把 app 二进制文件复制到这个镜像里面. 

注意这两个 COPY 都是 --from=builder , 这里的 builder 和第一阶段的 as builder 是一一对应的. 

最后把以上两个阶段的构建代码都复制到 Dockerfile 里面就结束了. 

### 打包镜像

配置文件好了, 剩下就是构建镜像了, 直接基于 Dockerfile 文件用 docker 打包即可! 

所以需要在安装好 docker 环境的机器上才能构建 docker 镜像. 

```bash
docker build -t gin-deom:v1 .
```

这里的 gin-deom:v1 就是你的镜像名字了, 我这里是随便写的, 实际生产中肯定是域名 + 镜像名字 + 版本号这样的格式. 