---
title: "MariaDB 修改字符编码格式"
date: 2021-11-26T15:58:03+08:00
draft: false
tags: ["MariaDB", "字符编码"]
---

## 修改配置文件

`vi /etc/my.cnf.d/server.conf`

```ini
[mysqld]
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'
skip-character-set-client-handshake=true
```

## 查询编码

```bash
mysql> SHOW VARIABLES WHERE Variable_name LIKE 'character_set_%' OR Variable_name LIKE 'collation%';

+--------------------------+-------------------+
| Variable_name            | Value             |
+--------------------------+-------------------+
| character_set_client     | utf8              |
| character_set_connection | utf8              |
| character_set_database   | latin1            |
| character_set_filesystem | binary            |
| character_set_results    | utf8              |
| character_set_server     | latin1            |
| character_set_system     | utf8              |
| collation_connection     | utf8_general_ci   |
| collation_database       | latin1_swedish_ci |
| collation_server         | latin1_swedish_ci |
+--------------------------+-------------------+
10 rows in set (0.00 sec)
```
<!--more-->