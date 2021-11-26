---
title: "CentOS添加swap分区"
date: 2021-11-26T16:25:26+08:00
draft: false
tags: ["Linux","CentOS","swap"]
---

```
/bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=1024
/sbin/mkswap /var/swap.1
/sbin/swapon /var/swap.1
```

[https://stackoverflow.com/questions/33299302/composer-update-failed-out-of-memory](https://stackoverflow.com/questions/33299302/composer-update-failed-out-of-memory)

Katiak's answer worked but I had to modify it. You will need 4 GB of free space for this to work on a Linux machine. Make sure you sudo the commands if you're not root:

```
/bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=4096
/sbin/mkswap /var/swap.1
/sbin/swapon /var/swap.1
```
<!--more-->
Composer takes alot of memory for certain repositories like Drupal.

Essentially, this creates 4 GB of Swap memory from the hard drive which the CPU can use to complete the composer command.

The original solution seems to have come from this Github thread but I could be mistaken:

[https://github.com/composer/composer/issues/7348#issuecomment-414178276](https://github.com/composer/composer/issues/7348#issuecomment-414178276)

To load the swap at boot add the following line in /etc/fstab

```
/var/swap.1 none swap sw 0 0
```

You may want to backup your fstab file just to be on the safe side.

To reclaim the swap space do this:

```
sudo swapoff -v /var/swap.1
sudo rm /var/swap.1
```

If you get a message like this after turning on the swap...

swapon: /var/swap.1: insecure permissions 0644, 0600 suggested.

...change the permission if appropriate

```
sudo chmod 600 /var/swap.1
```