---
title: "Composer 国内镜像"
date: 2021-11-26T16:20:19+08:00
draft: false
tags: ["composer","mirrors"]
---

# 阿里云

全局配置（推荐）

所有项目都会使用该镜像地址：

```
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```

取消配置：

```
composer config -g --unset repos.packagist
```

项目配置

仅修改当前工程配置，仅当前工程可使用该镜像地址：

```
composer config repo.packagist composer https://mirrors.aliyun.com/composer/
```

取消配置：

```
composer config --unset repos.packagist
```
<!--more-->
调试

`composer` 命令增加 `-vvv` 可输出详细的信息，命令如下：

```
composer -vvv require alibabacloud/sdk
```

## 腾讯云

切换镜像指向

```
composer config -g repos.packagist composer https://mirrors.cloud.tencent.com/composer/
```