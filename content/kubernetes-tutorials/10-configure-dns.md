---
title: "10. 配置 DNS"
date: 2019-04-03T20:17:11+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - dns
  - kubernetes-tutorials-1
---

<!--more-->

## 安装

使用 Dnsmasq 程序来实现内部域名解析

```shell
# 安装
$ sudo yum install -y dnsmasq

# 启动
$ sudo systemctl --now enable dnsmasq
```

## 配置

在 /etc/dnsmasq.conf 文件中增加所需要解析的域名

```python
# /etc/dnsmasq.conf
address=/platform.tcce.local/10.172.23.166
address=/dashboard.tcce.local/10.172.23.166
```

重启 Dnsmasq 服务

```shell
$ sudo systemctl restart dnsmasq
```

## 配置本地设备

修改本地网络中的 DNS 配置，指向 DNS 服务器 IP 地址：

![Mac DNS Configuration](configure.png)
