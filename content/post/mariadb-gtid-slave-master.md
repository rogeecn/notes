---
title: "MariaDB基于GTID的主从复制"
date: 2021-11-26T16:15:59+08:00
draft: false
tags: ["MariaDB","主从"]
---

MySQL复制有两种方式：

- 基于日志点的复制

- 基于GTID的复制

两种方式都依赖于MySQL二进制日志。

```ini
# 二进制日志格式
binlog-format=STATEMENT|ROW|MIXED
# ROW格式下日志的级别
binlog-row-image=full|minimal|noblob
```
<!--more-->
### 基于GTID的复制

基于GTID的复制是MySQL5.6后的一种新的复制方式。

> From MariaDB 10.0.2, global transaction ID is enabled automatically.
MariaDB从10.0.2开始默认开启GTID

**GTID即全局事务ID**，其保证了每个在Master上提交的事务在复制集群中可以生成一个唯一的ID。

GTID由3段组成，domain ID-server ID-sequence number

- domain ID：单master的环境下， domain id默认值0；多源复制是，domain id区分多个源；

- cserver ID：各master节点的sever-id，下面的配置文件有此项；

- sequence number：事务序列号

### 配置基于GTID复制的详细步骤：

在主DB上建立复制账号 

```bash
MariaDB [(none)]> create user 'slave'@'3.112.77.51' identified by 'slave_password';
Query OK, 0 rows affected (0.00 sec)
```

授予复制权限

```C#
MariaDB [(none)]> grant replication slave on *.* to 'slave'@'3.112.77.51';
Query OK, 0 rows affected (0.00 sec)
```

主服务器参数配置

```ini
# ************主服务器*************
[mysqld]
# 服务器id
server-id = 1000
# 二进制日志文件格式
binlog-format=ROW
# ROW格式下日志的级别
binlog-row-image=minimal
# 此两项为打开从服务器崩溃二进制日志功能，信息记录在事物表而不是保存在文件
master-info-repository=TABLE
relay-log-info-repository=TABLE
# 二进制日志，后面指定存放位置。如果只是指定名字，默认存放在/var/lib/mysql下
log-bin = mariadb-bin
```

从服务器参数配置

```ini
# ************从服务器*************
# 服务器id（在集群中需要唯一）
server_id = 2001
# binlog日志文件（如果只有文件名，则保存在/usr/lib/mysql目录）
relay_log = mysql-relay-bin
# 开启gtid模式（需要同时下面两个） 默认开启的
# gtid_mode = on
# 强制gtid一致性（默认开启）
# enforce-grid-consistency
# 从服务器中记录binlog日志
log-slave-updates = on
read_only = on
master_info_repository = TABLE
relay_log_info_repository = TABLE
```

**初始化数据：备份主DB、还原到从DB**

这里需要注意下，第1、2步在主DB创建用于复制的账号，这里我们备份、还原所有的库，所以会将用户也还原到从DB。

```Bash
# 使用mysqldump备份数据
mysqldump --single-transaction --routines --all-databases > dump.sql;
# 或使用xtrabackup
xtrabackup --slave-info ...
# 还原
mysql -uroot -p < all.sql
```

**启动基于GTID的复制**

```bash
# 在备份文件dump.sql中显示了GTID值，也可以在主库中查看变量
mysql> set global gtid_slave_pos='0-1-148';
mysql> change master to master_host = 'mysql.dns.ipao.cloud',master_user = 'slave',master_password = 'slave_password',master_use_gtid = slave_pos;
# 启动主从复制
mysql> start slave;
```

**检查主从复制状态**

```bash
MariaDB [(none)]> show slave status\G show processlist;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.104
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mariadb-bin.000002
          Read_Master_Log_Pos: 993
               Relay_Log_File: mariadb-relay-bin.000002
                Relay_Log_Pos: 644
        Relay_Master_Log_File: mariadb-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 993
              Relay_Log_Space: 944
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1000
               Master_SSL_Crl:
           Master_SSL_Crlpath:
                   Using_Gtid: Slave_Pos
                  Gtid_IO_Pos: 0-1000-4
      Replicate_Do_Domain_Ids:
  Replicate_Ignore_Domain_Ids:
                Parallel_Mode: conservative
1 row in set (0.00 sec)

+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+----------+
| Id | User        | Host      | db   | Command | Time | State                                                                       | Info             | Progress |
+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+----------+
| 13 | system user |           | NULL | Connect | 4858 | Waiting for master to send event                                            | NULL             |    0.000 |
| 14 | system user |           | NULL | Connect | 4858 | Slave has read all relay log; waiting for the slave I/O thread to update it | NULL             |    0.000 |
| 15 | root        | localhost | NULL | Query   |    0 | init                                                                        | show processlist |    0.000 |
+----+-------------+-----------+------+---------+------+-----------------------------------------------------------------------------+------------------+----------+
3 rows in set (0.00 sec)
```

**优点：**方便故障转移；从库不会丢失主库上的任何修改。
**缺点：**故障处理比较复杂；对执行的SQL有一定的限制。

#### 主从复制无法解决的问题

- 无法分担写负载，但可以分担读负载；

- 无法实现自动故障转移和主从切换，但提供了高可用架构基础；

- 不提供读写分离功能。
