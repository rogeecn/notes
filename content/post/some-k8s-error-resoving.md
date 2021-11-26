---
title: "Kubernetes 相关问题"
date: 2021-11-26T15:33:03+08:00
draft: false
tags: ["Kubernetes", "k8s", "error"]
---
## K8s kubectl error：c-bash: _get_comp_words_by_ref: command not found

故障现象：

```bash
source <(kubectl completion bash)
```

运行kubectl tab时出现以下报错

```bash
[root@master bin]# kubectl c-bash: _get_comp_words_by_ref: command not found
```

解决方法：
<!--more-->
1.安装bash-completion

```bash
[root@master bin]# yum install bash-completion -y
```

2.执行bash_completion

```bash
[root@master bin]# source /usr/share/bash-completion/bash_completion
```

3.重新加载kubectl completion

```bash
[root@master bin]# source <(kubectl completion bash)
```

4.又能开心的用tab了

```bash
[root@master bin]# kubectl create clusterrolebinding 
```

## 解决kubectl get pods时报错 No resources found.

原因: 暂时没有找到，启动什么的无异常，网上搜索后初步判断:systemd 单元文件包含ServiceAccount许可控制器，需要指定所需的签名密钥，详见Gitbub。

解决方案
跳过ServiceAccount(未成功)
打开kube-apiserver配置文件:

```bash
sudo vim /etc/kubernetes/apiserver
```

找到如下代码

```bash
"KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
```

去掉ServiceAccount，保存退出。

重启该项服务

启动报错，此方案不通

配置ServiceAccount(成功)

生成密钥到kube配置文件目录

```bash
openssl genrsa -out /etc/kubernetes/serviceaccount.key 2048
```

打开kube-apiserver配置文件:

```bash
sudo vim /etc/kubernetes/apiserver
```

追加如下内容：

```bash
KUBE_API_ARGS="--service_account_key_file=/etc/kubernetes/serviceaccount.key"
```

打开controller-manager配置文件:

```bash
sudo vim /etc/kubernetes/controller-manager
```

重写如下内容

```bash
KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/etc/kubernetes/serviceaccount.key"
```

重启相关项服务：

```bash
sudo systemctl restart etcd kube-apiserver kube-controller-manager kube-scheduler kube-proxy kubelet
```

### 测试

键入:

```bash
sudo kubectl get pods
```

### 参考文章

- Pods test failures after master upgrade 0.19.3 → 0.21.2 [https://github.com/kubernetes/kubernetes/issues/11355#issuecomment-127378691](https://github.com/kubernetes/kubernetes/issues/11355#issuecomment-127378691)

- 解决k8s创建pod报错No API token found for service account “default”, retry after the token is automatically

[https://blog.csdn.net/lusyoe/article/details/79673058](https://blog.csdn.net/lusyoe/article/details/79673058)

- Error creating kubernetes pod: No API token found for service acc [https://linuxacademy.com/community/posts/show/topic/15747-error-creating-kubernetes-pod-no-api-token-found-for-service-acc](https://linuxacademy.com/community/posts/show/topic/15747-error-creating-kubernetes-pod-no-api-token-found-for-service-acc)

- kubenetes无法创建pod/创建RC时无法自动创建pod的问题 [https://blog.csdn.net/jinzhencs/article/details/51435020](https://blog.csdn.net/jinzhencs/article/details/51435020)

## "POD" with ImagePullBackOff: "Back-off pulling image "registry.access.redhat.com/rhel7/pod-infrastructure:latest""

```bash
yum install wget -y
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm
rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
```

## nodeport 无法访问

为了安全起见， docker 在 1.13 版本之后，将系统iptables 中 FORWARD 链的默认策略设置为 DROP，并为连接到 docker0 网桥的容器添加了放行规则。这里引用 moby issue#14041 中的描述：

> When docker starts, it enables net.ipv4.ip_forward without changing the iptables FORWARD chain default policy to DROP. This means that another machine on the same network as the docker host can add a route to their routing table, and directly address any containers running on that docker host.

For example, if the docker0 subnet is 172.17.0.0/16 (the default subnet), and the docker host’s IP address is 192.168.0.10, from another host on the network run:
bash
$ ip route add 172.17.0.0/16 via 192.168.0.10
$ nmap 172.17.0.0/16

The above will scan for containers running on the host, and report IP addresses & running services found.

To fix this, docker needs to set the FORWARD policy to DROP when it enables the net.ipv4.ip_forward sysctl parameter.

kubernetes 使用的 cni 插件会因此受影响（cni并不会在 FORWARD 链中生成相应规则），由此导致除 pod 所在 host 以外节点无法转发报文而访问失败。

### Solution

如果对安全要求较低，可将 FORWARD 链的默认规则设为 ACCEPT

```bash
iptables -P FORWARD ACCEPT
```

其它方法可参考:

kubernetes issues#40182

[https://github.com/kubernetes/kubernetes/issues/40182](https://github.com/kubernetes/kubernetes/issues/40182)

## 解决flannel下k8s pod及容器无法跨主机互通问题

这是由于linux还有底层的iptables，所以在node上分别执行：

```bash
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F
iptables -L -n
```

## k8s 1.13部署pod和node中无法ping同cluster ip

问题
在node上和pod中无法ping通cluster ip

节点之前的网络是kube-proxy管理的，检查kube-proxy 的配置

```bash
vim /lib/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=192.168.205.10 \
  --v=4 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

导致这个问题的配置项是 --proxy-mode

> --proxy-mode ProxyMode                         Which proxy mode to use: 'userspace' (older) or 'iptables' (faster) or 'ipvs' (experimental). If blank, use the best-available proxy (currently iptables).  If the iptables proxy is selected, regardless of how, but the system's kernel or iptables versions are insufficient, this always falls back to the userspace proxy.

解决方法

修改配置如下：

```ini
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=192.168.205.10 \
  --v=4 \
  --proxy-mode=ipvs \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

将 `--proxy-mode`指定成 `ipvs`模式。

详细问题原因还不明，反正这样改了后，重启kube-proxy后就可以ping同cluster ip了。
