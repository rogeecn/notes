---
title: "Nginx 微信图片反向代理"
date: 2021-11-26T16:17:42+08:00
draft: false
tags: ["Nginx","WeChat", "反射代理"]
---



```nginx
server {
    listen 80;
    server_name qimg.jdwan.com;

    access_log off;

    location / {
        resolver 114.114.114.114;

        set $proxy_to http:/$uri;
        proxy_set_header Referer http://mp.weixin.qq.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass $proxy_to;

        expires 365d;
    }
}
```

<!--more-->