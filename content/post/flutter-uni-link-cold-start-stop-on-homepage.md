---
title: "Flutter解决uni_link 应用冷启动只能停留在主页的问题"
date: 2021-11-26T15:32:02+08:00
draft: false
tags: ["Flutter", "uni_link"]
---

```dart
StreamSubscription _sub;

// jdmbest://goto
  Future<Null> initUniLinks() async {
    String initialLink;
    // Platform messages may fail, so we use a try/catch PlatformException.
    try {
      initialLink = await getInitialLink();
      print('initial scheme link: $initialLink');
      if (initialLink != null) {
        print('initialLink--$initialLink');
        final Uri _jumpUri = Uri.parse(initialLink.replaceFirst(
          'jdmbest://',
          'http://',
        ));

        print("Scheme Changed: " +
            _jumpUri.path.toString() +
            ", query : " +
            _jumpUri.queryParameters.toString());
        //  跳转到指定页面
        Routes.navigateTo(context, _jumpUri.path,
            params: _jumpUri.queryParameters);
      }
    } on PlatformException {
      initialLink = 'Failed to get initial link.';
    } on FormatException {
      initialLink = 'Failed to parse the initial link as Uri.';
    }

    _sub = getUriLinksStream().listen((Uri uri) {
      print("Scheme Changed: " +
          uri.path.toString() +
          ", query : " +
          uri.queryParameters.toString());
      Routes.navigateTo(context, uri.path, params: uri.queryParameters);
//    Routes.navigateTo(context, uri.path, params: uri.queryParameters);
    }, onError: (err) {
      print("jump error: " + err.toString());
      // Handle exception by warning the user their action did345not succeed
    });
  }
```

<!--more-->