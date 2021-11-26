---
title: "Flutter 获取webview中cookie存储的session"
date: 2021-11-26T16:06:24+08:00
draft: false
tags: ["Flutter", "Session", "Cookie"]
---

## 需求

由于最近要获取阿里系网站的信息，网站在登录模块做了大量的保护工作，我的需求是拿到登录之后的cookie以获得session字段的值，用来保持登录状态。所以想通过webview来获取登录的信息。

在flutter上面比较流行的一个webview插件为：`flutter_webview_plugin`

## flutter_webview_plugin 使用

### 1.在app的路由上面添加登录的页面

```dart
routes: {
    "/web_login":(_) => new WebviewScaffold(url: Config.login,
        withLocalStorage: true,
        hidden: true,
        userAgent: "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Mobile Safari/537.36",
        appBar: new AppBar(
          centerTitle: true ,
          title: new Text("登录"),
        ),
        initialChild: Container(
          child: new Center(
            child: new CupertinoActivityIndicator(animating: true),
          ),
        ),
    ),
}
```

注意：userAgent字段一定要加上，不然在ios下打开WKwebview会白屏
<!--more-->
### 2.初始化flutterWebviewPlugin，作为全局变量；并且在启动app时如果没登录就跳转到登录界面

```dart
final flutterWebviewPlugin = new FlutterWebviewPlugin();

if(_event.data.length == 0){
    print("nologin ");
    Navigator.of(context).pushNamed("/web_login");
    _webViewLogin();
}else{
    print("get events data successfully");
}
```

### 3.监听url的变化，监测到跳转到成功之后的界面后获取cookies然后保存到sp中

```dart
flutterWebviewPlugin.onUrlChanged.listen((String url){
    var _uri = Uri.parse(url);
    if(_uri.path == "/dashboard"){
    Navigator.of(context).pop();
    flutterWebviewPlugin.close();
    print("登陆成功");
    flutterWebviewPlugin.getCookies().then((Map<String,String> _cookies){
        print(_cookies);
        _saveCookie();
    });
    }
});
```

以为这样就大功告成，但是打印出来的cookie其实是不包含session字段的。

看一下getCookies()方法的源码：

```dart
Future<Map<String, String>> getCookies() async {
    final cookiesString = await evalJavascript('document.cookie');
    final cookies = <String, String>{};

    if (cookiesString?.isNotEmpty == true) {
        cookiesString.split(';').forEach((String cookie) {
        final split = cookie.split('=');
        cookies[split[0]] = split[1];
        });
    }

    return cookies;
}
```

通过js来获得cookie，大多数网站都会做跨域安全处理，这样基本上不可能会获得session的值，那就毫无意义。

## 改轮子

目前flutter 插件中并没有可以直接获取 `webview cookie` 插件，因此只能自己改轮子

`flutter_webview_plugin` 使用的是 `webview`和`wkwebview`;

### 修改插件的方式为：

Android Studio左侧 projet窗口 > External Libraries > Flutter Plugins > flutter_webview_plugin

## Android获取cookie实现：

### 1.修改`flutter_webview_plugin/android`文件夹下的 `WebviewManager.java`

新加一个方法：

```dart
void getAllCookies(MethodCall call, final MethodChannel.Result result){
    String url = call.argument("url");
    CookieManager cookieManager = CookieManager.getInstance();
    String cookieStr = cookieManager.getCookie(url);
    result.success(cookieStr);
}
```

### 2.修改flutter_webview_plugin/android文件夹下的FlutterWebviewPlugin.java

```dart
@Override
public void onMethodCall(MethodCall call, MethodChannel.Result result) {
    switch (call.method) {
        case "launch":
            openUrl(call, result);
            break;
        case "close":
            close(call, result);
            break;
        case "eval":
            eval(call, result);
            break;
        case "resize":
            resize(call, result);
            break;
        case "reload":
            reload(call, result);
            break;
        case "back":
            back(call, result);
            break;
        case "forward":
            forward(call, result);
            break;
        case "hide":
            hide(call, result);
            break;
        case "show":
            show(call, result);
            break;
        case "reloadUrl":
            reloadUrl(call, result);
            break;
        case "stopLoading":
            stopLoading(call, result);
            break;
        case "cleanCookies":
            cleanCookies(call, result);
            break;
        case "getAllCookies":
            getAllCookies(call,result);
                break;
        default:
            result.notImplemented();
            break;
    }
}
```

在`onMethodCall`方法上新增case的分支，用于判断使用的方法是`getAllCookies`

### 3.新增一个getAllCookies的方法

```dart
private void getAllCookies(MethodCall call, final MethodChannel.Result result){
    if (webViewManager != null){
        webViewManager.getAllCookies(call,result);
    }
}
```

## IOS获取Cookie的实现

### 1.修改`flutter_webview_plugin/ios/Classes`文件夹下的`FlutterWebviewPlugin.m`

```objective-c
- (void)handleMethodCall:(FlutterMethodCall*)call result:(FlutterResult)result {
    if ([@"launch" isEqualToString:call.method]) {
        if (!self.webview)
            [self initWebview:call];
        else
            [self navigate:call];
        result(nil);
    } else if ([@"close" isEqualToString:call.method]) {
        [self closeWebView];
        result(nil);
    } else if ([@"eval" isEqualToString:call.method]) {
        [self evalJavascript:call completionHandler:^(NSString * response) {
            result(response);
        }];
    } else if ([@"resize" isEqualToString:call.method]) {
        [self resize:call];
        result(nil);
    } else if ([@"reloadUrl" isEqualToString:call.method]) {
        [self reloadUrl:call];
        result(nil);
    } else if ([@"show" isEqualToString:call.method]) {
        [self show];
        result(nil);
    } else if ([@"hide" isEqualToString:call.method]) {
        [self hide];
        result(nil);
    } else if ([@"stopLoading" isEqualToString:call.method]) {
        [self stopLoading];
        result(nil);
    } else if ([@"cleanCookies" isEqualToString:call.method]) {
        [self cleanCookies];
    } else if ([@"back" isEqualToString:call.method]) {
        [self back];
        result(nil);
    } else if ([@"forward" isEqualToString:call.method]) {
        [self forward];
        result(nil);
    } else if ([@"reload" isEqualToString:call.method]) {
        [self reload];
        result(nil);
    }else if ([@"getAllCookies" isEqualToString:call.method]){
        [self getAllCookies:call completionHandler:^(NSString *cookies) {
              result(cookies);
          }];
    }else {
        result(FlutterMethodNotImplemented);
    }
}
```

添加一个`getAllCookies`的分支判断

### 2.添加一个方法

⚠️  仅支持ios11以上

```objective-c
- (void)getAllCookies:(FlutterMethodCall*)call
     completionHandler:(void (^_Nullable)(NSString * cookies))completionHandler {
    if (self.webview != nil) {
        NSString *url = call.arguments[@"url"];
        WKHTTPCookieStore *cookieStore = self.webview.configuration.websiteDataStore.httpCookieStore;
        [cookieStore getAllCookies:^(NSArray<NSHTTPCookie *> * _Nonnull cookies) {
            NSString *allCookies = @"";
            NSEnumerator *cookie_enum = [cookies objectEnumerator];
            NSHTTPCookie *temp_cookie;
            while (temp_cookie = [cookie_enum nextObject]) {
                NSString *temp = [NSString stringWithFormat:@"%@=%@;",[temp_cookie name],[temp_cookie value]];
                allCookies = [allCookies stringByAppendingString:temp];
            }
            completionHandler([NSString stringWithFormat:@"%@", allCookies]);
        }];
    } else {
        completionHandler(nil);
    }
}
```

## Dart修改

修改`flutter_webview_plugin/lib/src`文件夹下的`base.dart`

### 添加getAllCookies的方法

```dart
/// Get AllCookies
Future<String> getAllCookies(String url) async {
    final res = await _channel.invokeMethod('getAllCookies', {'url': url});
    return res;
}
```

使用

```dart
Future<Null> _getAllCookies() async {
    try {
        final String result = await flutterWebviewPlugin.getAllCookies("https://www.yuque.com/dashboard");
        print(result);
        cookie = result;
        _saveCookie();
        _getHttpData();
    } on PlatformException catch (e) {
        cookie = "";
    }
}
```

注意：getAllCookies必须在浏览器未关闭之前获取，否则会获取空值
