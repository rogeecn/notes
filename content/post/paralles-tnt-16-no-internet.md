---
title: "Paralles Tnt 16 No Internet"
date: 2021-12-09T18:07:15+08:00
draft: false
tags: ["MAC", "Parallels","TNT"]
---


## 一、网络初始化问题

打开访达，找到 `/Library/Preferences/Parallels/network.desktop.xml` 文件，拷贝出来修改后替换原文件（注意备份）。

修改方法：找到 `<UseKextless>-1</UseKextless>` 改为 `<UseKextless>0</UseKextless>`，保存替代（在第五行的位置）！

Parallels Desktop 16 TNT 版解决无法链接网络及不识别 USB 问题

## 二、USB 不识别问题

打开访达，找到 /Library/Preferences/Parallels/dispatcher.desktop.xml 文件，拷贝出来修改后替换原文件（注意备份）。

修改方法：找到 `<Usb>0</Usb>`  改为 `<Usb>1</Usb>`，保存替代！
<!-- more -->