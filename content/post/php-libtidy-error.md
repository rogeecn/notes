---
title: "MAC Php Libtidy 报错解决办法"
date: 2021-12-06T09:33:02+08:00
draft: false
tags: ["PHP","composer"]
---

今天使用 `composer` 安装一个扩展包时`mac`报如下问题：

```bash
>> composer require laravelista/comments
dyld: Library not loaded: /usr/local/opt/tidy-html5/lib/libtidy.5.dylib
  Referenced from: /usr/local/bin/php
  Reason: image not found
[1]    33575 abort      composer require laravelista/comments
➜  MedicalCMS git:(master) which php                           
```

解决方案为重新安装此扩展：

```bash
brew reinstall tidy-html5
```
<!--more-->

但是导致此问题的原因没有找到！