---
title: "5. 安装 Consul"
date: 2019-04-02T22:24:53+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - consul
  - kubernetes-tutorials-1
---

<!--more-->

## 准备配置文件
```shell
$ curl -o consul.yml https://raw.githubusercontent.com/helm/charts/master/stable/consul/values.yaml
```

修改配置如下：
```yaml
ImageTag: "1.4.4"
StorageClass: "rook-ceph-block"
```

## 安装：
```shell
$ helm install --name consul -f consul.yml stable/consul
```

## 验证安装：
```shell
$ kubectl get service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                            AGE
consul       ClusterIP      None            <none>        8500/TCP,8400/TCP,8301/TCP,8301/UDP,8302/TCP,8302/UDP,8300/TCP,8600/TCP,8600/UDP   11m
consul-ui    NodePort       10.105.230.25   <none>        8500:30664/TCP                                                                     11m
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP
```

可以看到 consul-ui 暴露端口至 `30664`，使用浏览器打开：

![Consul-UI](consul-ui.png)
