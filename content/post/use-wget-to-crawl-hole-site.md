---
title: "使用wget命令爬取整站"
date: 2021-11-26T15:51:30+08:00
draft: false
tags: ["wget","Linux"]
---

## 命令
```
wget -c -r -npH -k -nv -user-agent="Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.104 Safari/537.36 Core/1.53.4482.400 QQBrowser/9.7.13001.400" http://www.baidu.com
```
<!--more-->
## 参数说明

```
-c ：断点续传

-r：递归下载

-np：递归下载时不搜索上层目录

-nv：显示简要信息

-nd：递归下载时不创建一层一层的目录,把所有文件下载当前文件夹中

-p：下载网页所需要的所有文件(图片,样式,js文件等)

-H：当递归时是转到外部主机下载图片或链接

-k：将绝对链接转换为相对链接,这样就可以在本地脱机浏览网页了

-L：只扩展相对连接，该参数对于抓取指定站点很有用，可以避免向宿主主机
```

## 启用地址伪装

```bash
-user-agent="Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/53.0.2785.104 Safari/537.36 Core/1.53.4482.400 QQBrowser/9.7.13001.400"
```

