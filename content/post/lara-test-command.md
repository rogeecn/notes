---
title: "Laravel 测试命令"
date: 2021-11-26T15:42:07+08:00
draft: false
tags: ["php","laravel", "bash", "test"]
---

复制下面内容到　`/usr/local/bin/lara-test`

```bash
#!/bin/bash

if [ $# == 0 ] ; then
    echo "Usage: lara-test [suite] [filter]"
    exit
fi

cd `pwd`

COMMAND="php artisan test"
SUITE=$1
FILTER=$2

if [ $# == 1 ] ; then
    $COMMAND --testsuite $SUITE
    exit
fi

if [ $# == 2 ] ; then
    $COMMAND --testsuite $SUITE --filter $FILTER
    exit
fi

shift
shift
$COMMAND --testsuite $SUITE --filter $FILTER $@
```
<!--more-->
给执行权限  

```bash
vi /usr/local/bin/lara-test
chmod +x /usr/local/bin/lara-test
```
