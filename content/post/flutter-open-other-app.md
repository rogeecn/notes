---
title: "Flutter 使用网页唤醒 APP"
date: 2021-11-26T16:04:41+08:00
draft: false
tags: ["Flutter"]
---

这里只介绍安卓的端的用法

## 修改配置文件

通过 `host` 和 `scheme` 来完成对 App 应用的匹配

文件：AndroidManifest.xml 在 activity 块下添加

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:host="splash" android:scheme="vchao" />
</intent-filter>
```
<!--more-->
测试可用性

WEB端通过链接就能打开了

```html
<a href="vchao://splash/?arg0=marc&arg1=xie">直接打开</a>
<a href="vchao://splash/?path=home&arg1=xie">打开首页</a>
<a href="vchao://splash/?path=anecdotal&arg1=xie">打开朋友圈</a>
```

## 关于参数的处理

Flutter 需要添加依赖 `uni_links` 包

```
uni_links: ^0.2.0
```

在底部渲染tab导航的页面注册监听方法，来接收跳转链接

```dart
initState() {
    super.initState();
    initUniLinks();
}

StreamSubscription _sub;
Future<Null> initUniLinks() async {
    print('------获取参数--------');
    // Platform messages may fail, so we use a try/catch PlatformException.
        _sub = getLinksStream().listen((String link) {
        Map params = Utils.formateUrl(link);
        String path = params['path'];
        print('path');
        print(path);
        print(context);
        if(path.isNotEmpty){
            RouterUtil.push(context,path,'');
        }
        print('---213---');
        print(link);
        // Parse the link and warn the user, if it is not correct
    }, onError: (err) {
        // Handle exception by warning the user their action did not succeed
    });
}
```

根据路径就能跳转到不同的页面了

## 使用 ADB 命令行功能测试

```bash
adb shell 'am start -W -a android.intent.action.VIEW -c android.intent.category.BROWSABLE -d "unilinks://example.com/path/portion/?uid=123&token=abc"'
```