---
title: "防止网站页面被其他网站iframe引用方法"
date: 2021-11-26T10:16:38+08:00
draft: false
tags: ["网站", "iframe"]
categories: ["网站开发","网站技巧"]

---

两种解决方案：

## 1、在响应头里加一个X-Frame-Options

其取值有三种，大部分浏览器都支持：

- DENY：浏览器拒绝当前页面加载任何Frame页面
- SAMEORIGIN：frame页面的地址只能为同源域名下的页面
- ALLOW-FROM origin：origin为允许frame加载的页面地址

这样被不同源的页面以iframe包含时就不会显示了
<!--more-->
### Apache 配置方法

在站点配置文件 `http.conf` 中添加如下配置，限制只有站点内(同源)的页面才可以使用 iframe 进行嵌入

```apache
Headaer always append X-Frame-Options "SAMEORIGIN"
```

修改完成后重启 `Apache` 使配置生效。
> 此配置对 __IBM HTTP Server__ 同样适用

如果同一 `Apache` 服务器上有多个虚拟网站（`VirtualHost`），只想针对同一个站点进行配置，可以修改 `.htaccess` 文件，添加如下内容：

```apache
Headaer always append X-Frame-Options "SAMEORIGIN"
```

### Nginx 配置方法

修改 `ngin.conf` 文件，添加如下配置：

```nginx
server{
    listen 80:
    server_name localhost;

    add_header X-Frame-Options "SAMEORIGIN"; # 这一行是新添加的
}
```

修改完成后测试配置并重载

```bash
# 测试配置
nginx -t 
# 重载配置
nginx -s reload 
```

## 2、使用 Javascript 脚本进行限制

在自己的页面中写入如下js脚本中的一个即可：

```js
if (window != window.top) {
    window.top.location.replace(window.location)
    // 这是直接代替外窗，你也可以干别的
}

if (top != self) {
    top.location = self.location;
}

if (self == top) {
    var theBody = document.getElementsByTagName('body')[0];
    theBody.style.display = "block";
} else {
    top.location = self.location;
}
```
