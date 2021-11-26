---
title: "解决 Flutter Android 添加scheme 后安装没有图标的问题"
date: 2021-11-26T15:29:48+08:00
draft: false
tags: ["Flutter", "Android", "Scheme", "Icon"]
---
```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>

<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:host="goto" android:scheme="jdmbest" />
</intent-filter>
```

<!--more-->