---
title: "Laravel Artisan 命令任意执行"
date: 2021-11-26T15:44:17+08:00
draft: false
tags: ["laravel","php","bash","artisan"]
---
只要在 laravel 项目根目录下即可运行 ,不需要再敲烦人的 `./` 前缀了。

```bash
#!/bin/bash

cd `pwd`
php artisan $@
```

<!--more-->