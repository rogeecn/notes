---
title: "Flutter 无法访问HTTP服务"
date: 2021-11-26T16:05:36+08:00
draft: false
tags: ["FLutter","Netword"]
---

## Android无法访问http

其实这本身不是Flutter的问题，但在开发中经常遇到，在Android Pie版本及以上和IOS 系统上默认禁止访问http，主要是为了安全考虑。

### Android解决办法：

在`./android/app/src/main/AndroidManifest.xml`配置文件中`application`标签里面设置`networkSecurityConfig`属性:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:networkSecurityConfig="@xml/network_security_config">
         <!-- ... -->
    </application>
</manifest>
```

在`./android/app/src/main/res`目录下创建xml文件夹（已存在不用创建），在xml文件夹下创建`network_security_config.xml`文件，内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```
<!--more-->
## IOS无法访问http

在`./ios/Runner/Info.plist`文件中添加如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    ...
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <true/>
    </dict>
</dict>
</plist>
```