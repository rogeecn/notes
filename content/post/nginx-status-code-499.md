---
title: "Nginx的 HTTP 499 状态码问题"
date: 2021-12-03T10:08:53+08:00
draft: false
tags: ["nginx", "网站"]
---

## 1、前言

今天在处理一个客户问题，遇到 Nginx access log 中出现大量的 `499` 状态码。实际场景是：客户的域名通过 cname 解析到我们的 Nginx 反向代理集群上来，客户的 Web 服务是由一个负载均衡提供外网 IP 进行访问，负载均衡后面挂了多个内网 web 站点业务服务器。出现的访问日志如下所示：

![img](305504-20170625232821913-1629695159.png)

## 2、处理方法

499 错误是什么？让我们看看 NGINX 的源码中的定义：

```c
ngx_string(ngx_http_error_495_page), /* 495, https certificate error */
ngx_string(ngx_http_error_496_page), /* 496, https no certificate */
ngx_string(ngx_http_error_497_page), /* 497, http to https */
ngx_string(ngx_http_error_404_page), /* 498, canceled */
ngx_null_string,                    /* 499, client has closed connection */
```

可以看到，499对应的是 “client has closed connection”。这很有可能是因为服务器端处理的时间过长，客户端“不耐烦”了。

测试nginx发现如果两次提交post过快就会出现499的情况，看来是nginx认为是不安全的连接，主动拒绝了客户端的连接.

## 解决方案

在google上搜索到一英文论坛上有关于此错误的**解决方法**：

```nginx
proxy_ignore_client_abort on;
```
就是说要配置参数 `proxy_ignore_client_abort on;`表示代理服务端不要主要主动关闭客户端连接。
重启nginx,问题果然得到解决。只是安全方面稍有欠缺，但比总是出现找不到服务器好多了。
