---
title: "2. 安装 Helm"
date: 2019-04-02T22:08:08+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - helm
  - kubernetes-tutorials-1
---

<!--more-->

## 下载
科学上网下载适用于服务器的二进制压缩包：

`https://kubernetes-helm.storage.googleapis.com/helm-v2.13.1-linux-amd64.tar.gz`

传输至服务器上：

```shell
$ scp helm-v2.13.1-linux-amd64.tar.gz t1:/opt/kubernetes
```

## 解压并安装

```shell
$ tar zxvf helm-v2.13.1-linux-amd64.tar.gz
$ sudo mv linux-amd64/helm /usr/local/bin/
$ sudo chmod +x /usr/local/bin/helm
```

```shell
# 查看版本
$ helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Error: could not find tiller
```

## 初始化集群

```shell
$ helm init
Creating /home/app/.helm
Creating /home/app/.helm/repository
Creating /home/app/.helm/repository/cache
Creating /home/app/.helm/repository/local
Creating /home/app/.helm/plugins
Creating /home/app/.helm/starters
Creating /home/app/.helm/cache/archive
Creating /home/app/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/app/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

再次查看版本：

```shell
$ helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```

可以看到 Server 的版本成功获取。

## 初始化账号和权限

```shell
$ kubectl --namespace kube-system create serviceaccount tiller

$ kubectl create clusterrolebinding tiller-cluster-rule \
 --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

$ kubectl --namespace kube-system patch deploy tiller-deploy \
 -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
