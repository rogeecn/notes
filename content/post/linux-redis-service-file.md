---
title: "Linux Redis Service 配置文件"
date: 2021-11-26T15:49:23+08:00
draft: false
tags: ["Linux","Redis"]
---

```bash
vi /usr/lib/systemd/system/redis.service
```

```ini
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/redis-server /etc/redis.conf --supervised systemd
ExecStop=/usr/libexec/redis-shutdown
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
```

<!--more-->