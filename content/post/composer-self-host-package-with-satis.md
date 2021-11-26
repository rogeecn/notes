---
title: "用 Satis 处理私有资源包"
date: 2021-11-26T16:20:59+08:00
draft: false
tags: ["composer","satis"]
---

Satis 是 `composer`官方推出的一款私有资源包托管工具。项目地址：[https://github.com/composer/satis](https://github.com/composer/satis)

## 安装方式

### 本地安装

```bash
composer create-project composer/satis:dev-master
```
<!--more-->

### Docker 镜像 安装

```php
docker run --rm --init -it \
  --user $(id -u):$(id -g) \
  --volume $(pwd):/build \
  --volume "${COMPOSER_HOME:-$HOME/.composer}:/composer" \
  composer/satis build <configuration-file> <output-directory>
```

## 使用方式

配置 `satis.json`文件

```json
{
    "name": "satis/repo",
    "homepage": "http://packages.example.org",
    "repositories": [
        // 一行一个写入 gitlab 项目地址
        { "type": "vcs", "url": "git@git-biz.qianxin-inc.cn:seccloud/ssmp-composer-eolinker.git" },
    ],
    "require-all": true
}
```

运行命令生成包缓存, 将会在`packages`目录生成索引文件

```bash
php bin/satis build satis.json packages
```

生成完成后启动 WEB 服务器

```php
php -S localhost:8080 -t ./packages
```

配置项目`composer.json`文件

```json
{
    "repositories": [ { "type": "composer", "url": "http://localhost:8080/" } ],
    "require": {
        "company/package": "1.2.0", 
    }
}
```

更新项目依赖

```
composer update -vvv
```

## satis.json

```
{
  "name": "seccloud/satis",
  "homepage": "http://satis.dev.local",
  "repositories": [
    {
      "type": "vcs",
      "url": "git@git-biz.qianxin-inc.cn:seccloud/ssmp-composer-eolinker.git"
    },
    {
      "type": "vcs",
      "url": "git@git-biz.qianxin-inc.cn:seccloud/ssmp-yii2-jobs.git"
    },
    {
      "type": "vcs",
      "url": "git@git-biz.qianxin-inc.cn:seccloud/ssmp-swoole-job.git"
    }
  ],
  "require": {
    "ssmp/swoole-jobs": "*",
    "ssmp/yii2-swoole-jobs": "*",
    "ssmp/php-annotation-hera": "*"
  },
  "archive": {
    "directory": "dist",
    "format": "zip",
    "prefix-url": "http://localhost:8080",
    "skip-dev": true
  },
  "require-all": false
}
```