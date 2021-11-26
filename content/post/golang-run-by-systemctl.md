---
title: "使用 systemctl 启动 Golang 应用"
date: 2021-11-26T15:50:15+08:00
draft: false
tags: ["Linux", "Golang","systemctl"]
---

```ini
[Unit]
Description=jdm apk building service
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/opt/builder/builder serve --config /opt/builder/etc/builder.toml
Type=simple
User=root
Group=root
RuntimeDirectory=/opt/builder
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target

```