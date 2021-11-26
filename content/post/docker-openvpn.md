---
title: "Docker Openvpn"
date: 2021-11-26T16:33:49+08:00
draft: false
tags: ["Docker","OpenVPN"]
---


## 安装 Docker-CE

```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

## 拉取镜像

```Bash
docker pull kylemanna/openvpn
```
<!--more-->
## 生成配置文件

```bash
docker run -v /opt/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://$hostname
```

## 生成密钥文件

```bash
docker run -v /opt/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
```

输入私钥密码（输入时是看不见的）：  
Enter PEM pass phrase:12345678  
再输入一遍  
Verifying - Enter PEM pass phrase:12345678  
输入一个CA名称（我这里直接回车）  
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:  
输入刚才设置的私钥密码（输入完成后会再让输入一次）  
Enter pass phrase for /etc/openvpn/pki/private/ca.key:12345678

## 生成客户端证书

这里的s360改成你想要的名字

```bash
docker run -v /opt/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full s360 nopass
```

## 导出客户端配置

```bash
mkdir -p /opt/openvpn/conf
docker run -v /opt/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient s360 > /opt/openvpn/conf/s360.ovpn
```

## 启动OpenVPN服务

```bash
docker run --name openvpn -v /opt/openvpn:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
```

## 保存防火墙规则

```bash
iptables-save > /etc/sysconfig/iptables

```

## 设置防火墙

关闭firewalld防火墙，关闭开机自启

```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

安装iptables防火墙，设置开机自启

```bash
yum -y install iptables-services net-tools
systemctl enable iptables.service
```

编辑防火墙配置

```bash
vi /etc/sysconfig/iptables
```

在最后COMMIT前添加以下规则

```bash
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
```

下面是一个完整的示例（这里只是个示例，根据自身情况对防火墙进行调整）

```bash
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [3:228]
:POSTROUTING ACCEPT [3:228]
:DOCKER - [0:0]
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.17.0.2/32 -d 172.17.0.2/32 -p udp -m udp --dport 1194 -j MASQUERADE
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i docker0 -p udp -m udp --dport 1194 -j DNAT --to-destination 172.17.0.2:1194
COMMIT
*filter
:INPUT ACCEPT [60:4900]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [50:4784]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p udp -m udp --dport 1194 -j ACCEPT
-A DOCKER-ISOLATION -j RETURN
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

## 重启防火墙

```bash
systemctl restart iptables
```

## 将登录的证书下载到本地

```bahs
yum install lrzsz -y
sz /data/openvpn/conf/rogee.ovpn
```

openvpn windows客户端配置

openvpn客户端下载：[https://openvpn.net/downloads/openvpn-connect-v3-windows.msi](https://openvpn.net/downloads/openvpn-connect-v3-windows.msi) 

在openvpn的安装目录下，有个config目录，将服务器上的rogee.ovpn，放在该目录下，运行OpenVPN GUI，右键 rogee 连接connect

最后验证，打开百度输入ip

## **附录：**

为了方便使用，写两个删除和创建脚本

### openvpn删除用户脚本

```bash
#!/bin/bash
read -p "Delete username: " DNAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa revoke $DNAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa gen-crl
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn rm -f /etc/openvpn/pki/reqs/"$DNAME".req
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn rm -f /etc/openvpn/pki/private/"$DNAME".key
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn rm -f /etc/openvpn/pki/issued/"$DNAME".crt
docker restart openvpn
```

### 创建用户

```bash
#!/bin/bash
read -p "please your username: " NAME
docker run -v /data/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full $NAME nopass
docker run -v /data/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $NAME > /data/openvpn/conf/"$NAME".ovpn
docker restart openvpn
```

### 一键安装

```bash
hostname=`hostname`
user=s360

mkdir -p /opt/openvpn/conf

docker pull kylemanna/openvpn
docker run -v /opt/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://$hostname
docker run -v /opt/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki nopass

# 这里要输入密码了

docker run -v /opt/openvpn:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full $user nopass
docker run -v /opt/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $user > /opt/openvpn/conf/$user.ovpn
docker run --name openvpn -v /opt/openvpn:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
```
