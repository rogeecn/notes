---
title: "go build 移除路径信息"
date: 2021-11-26T16:26:59+08:00
draft: false
tags: ["Golang"]
---


```bash
CGO_ENABLED=0 go build -v -a -ldflags '-s -w' -gcflags="all=-trimpath=${PWD}" -asmflags="all=-trimpath=${PWD}"
```

<!--more-->