---
title: "11. 配置集群状态监控"
date: 2019-04-03T20:22:24+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - promethues
  - grafana
  - kubernetes-tutorials-1
---

<!--more-->

## 准备

接下来的部署中，有一个镜像位于谷歌服务器，所以需要手动拉取和打标签：

```shell
$ docker pull gcr.azk8s.cn/google_containers/addon-resizer-amd64:2.1
$ docker tag gcr.azk8s.cn/google_containers/addon-resizer-amd64:2.1 gcr.io/google-containers/addon-resizer-amd64:2.1
```

## 安装

```shell
$ git clone https://github.com/coreos/prometheus-operator.git
```

查看最新的 release 版本：https://github.com/coreos/prometheus-operator/releases

这里使用的是最新的 v0.29.0：
```shell
$ cd prometheus-operator
$ git checkout v0.29.0
```

```shell
$ cd prometheus-operator/contrib/kube-prometheus
$ kubectl create -f manifest
```

## 验证

在 Dashboard 上观察所有部署都完成之后，暴露 Grafana 端口：

```shell
$ kubectl get service -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       ClusterIP   10.97.249.194   <none>        9093/TCP            44m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   27m
grafana                 ClusterIP   10.102.17.235   <none>        3000/TCP            44m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   44m
node-exporter           ClusterIP   None            <none>        9100/TCP            44m
prometheus-adapter      ClusterIP   10.105.2.73     <none>        443/TCP             44m
prometheus-k8s          ClusterIP   10.102.30.255   <none>        9090/TCP            44m
prometheus-operated     ClusterIP   None            <none>        9090/TCP            27m
prometheus-operator     ClusterIP   None            <none>        8080/TCP            44m
```

```shell
kubectl edit service grafana -n monitoring
# 将 spec.type 改为 NodePort 保存生效
```

再次查看服务：
```shell
$ kubectl get service -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       ClusterIP   10.97.249.194   <none>        9093/TCP            44m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   28m
grafana                 NodePort    10.102.17.235   <none>        3000:44550/TCP      44m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   44m
node-exporter           ClusterIP   None            <none>        9100/TCP            44m
prometheus-adapter      ClusterIP   10.105.2.73     <none>        443/TCP             44m
prometheus-k8s          ClusterIP   10.102.30.255   <none>        9090/TCP            44m
prometheus-operated     ClusterIP   None            <none>        9090/TCP            27m
prometheus-operator     ClusterIP   None            <none>        8080/TCP            44m
```

Grafana 的外部端口为 44550，使用浏览器访问：

使用默认用户名密码：admin admin 登录：

![Grafana Dashboard](grafana-dashboard.png)

集群状态：

![Cluster Info](cluster-info.png)

Kubernetes 节点状态：

![节点状态](node-status.png)
