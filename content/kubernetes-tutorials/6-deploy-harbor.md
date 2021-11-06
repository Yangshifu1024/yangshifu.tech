---
title: "6. 部署 Harbor"
date: 2019-04-02T22:29:54+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - docker
  - harbor
  - kubernetes-tutorials-1
---

<!--more-->

## 准备配置文件

```shell
$ git clone https://github.com/goharbor/harbor-helm
$ cd harbor-helm
```

修改 `values.yaml` 如下：
```yaml
# 仅展示有修改的部分
expose:
  type: ingress
  ingress:
    host: harbor.tcce.local
externalURL: https://harbor.tcce.local
persistence:
  enabled: true
  persistentVolumeClaim:
    registry:
      storageClass: "rook-ceph-block"
    chartmuseum:
      storageClass: "rook-ceph-block"
    jobservice:
      storageClass: "rook-ceph-block"
    database:
      storageClass: "rook-ceph-block"
    redis:
      storageClass: "rook-ceph-block"
imagePullPolicy: IfNotPresent

logLevel: debug
# admin 用户的初始化密码
harborAdminPassword: "[密码]"
```

## 测试安装结果
访问 https://harbor.tcce.local

![Harbor](harbor.png)

## 配置所有节点

```shell
$ sudo vim /etc/docker/deamon.json
```
增加如下内容：
```json
"insecure-registries": [
    "harbor.tcce.local"
]
```
