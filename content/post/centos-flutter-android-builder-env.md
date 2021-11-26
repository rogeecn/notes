---
title: "CentOS7 Flutter Android 环境构建"
date: 2021-11-26T16:03:43+08:00
draft: false
tags: ["Flutter", "Android", "CentOS"]
---

## Flutter

### 下载 Flutter SDK 

  [https://flutter.cn/docs/development/tools/sdk/releases?tab=linux](https://flutter.cn/docs/development/tools/sdk/releases?tab=linux)

### 配置环境变量

```bash
export PATH=$PATH:/root/flutter/bin
```
<!--more-->
### 二进制预先下载

```bash
flutter precache
```

## JAVA

下载地址：[https://www.oracle.com/java/technologies/javase-jdk8-downloads.html](https://www.oracle.com/java/technologies/javase-jdk8-downloads.html)

### 下载 Android SDK

[https://developer.android.com/studio/#downloads](https://developer.android.com/studio/#downloads)

下载后解压

```bash
unzip android-sdk.zip
```

### 配置环境变量

```bash
export ANDROID_HOME="/root/android-sdk”
export PATH="ANDROID_HOME/tools:ANDROID_HOME/tools/bin:ANDROID_HOME/platform-tools:PATH"
```

## 下载环境

查询SDK 版本环境相关数据 替换 ${ANDROID_COMPILE_SDK} 版本信息[https://developer.android.com/studio/releases/platforms?hl=zh-cn](https://developer.android.com/studio/releases/platforms?hl=zh-cn)

```bash
export ANDROID_HOME=PWD/android-sdk 
export ANDROID_COMPILE_SDK=26 yes | sdkmanager --sdk_root={ANDROID_HOME} --licenses sdkmanager --sdk_root={ANDROID_HOME} "platform-tools" "platforms;android-{ANDROID_COMPILE_SDK}" >/dev/null
```

### 编译APK

```bash
flutter build apk --release
```

## 问题整理

1. flutter -version 会显示 0.0.0-unknown 的版本信息

  需要升级下 git 工具的版本即可识别。