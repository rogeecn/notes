---
title: "Golang 在 Mac、Linux、Windows 下如何交叉编译"
date: 2021-11-26T16:01:14+08:00
draft: false
tags: ["Golang", "交叉编译"]
---

**交叉编译不支持 CGO 所以要禁用它**

## 如何编译

### Mac 下编译 Linux 和 Windows 64位可执行程序

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```
<!--more-->
### Linux 下编译 Mac 和 Windows 64位可执行程序

```bash
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go
```

### Windows 下编译 Mac 和 Linux 64位可执行程序

```bash
SET CGO_ENABLED=0
SET GOOS=darwin
SET GOARCH=amd64 go build main.go
SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build main.go
```

## GOOS：目标平台的操作系统

- darwin

- freebsd

- linux

- windows

## GOARCH：目标平台的体系架构

- 386

- amd64

- arm

上面的命令编译 64 位可执行程序，你当然应该也会使用 386 编译 32 位可执行程序