---
title: "4. 安装 MySQL"
date: 2019-04-02T22:20:49+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - mysql
  - kubernetes-tutorials-1
---

<!--more-->

## 安装
```shell
$ helm install --name mysql-57 --set mysqlRootPassword=root,persistence.storageClass=rook-ceph-block stable/mysql --debug
```

注意这里的 `persistence.storageClass` 要与 `StorageClass` 的 `name` 一致。

## 查看 Service
```shell
$ kubectl get service
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    19h
mysql-57     ClusterIP   10.97.14.175   <none>        3306/TCP   14m
```
可以看到 MySQL 服务以及启动。

## 暴露端口

```shell
$ kubectl edit service mysql-57
```

修改 `spec.type` 从 `ClusterIP` 为 `LoadBalancer`；新增 `spec.ports.nodePort = 32306`，保存生效。

## root 用户远程登录（测试环境）

```shell
# 查看 MySQL 所在 Pod
$ kubectl get pod | grep mysql
mysql-57-865cd8658-tc24x   1/1     Running             0          18m
```

```shell
# 进入 Pod 执行命令：
$ kubectl exec -it mysql-57-865cd8658-tc24x bash

# 进入 Pod 后连接 MySQL
$ mysql -uroot

# 进入 MySQL 后执行
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'root';
```

## Pod 间连接测试

```shell
$ kubectl run mysql-client-$RANDOM --rm -it --image=mysql:5.7.14 -- mysql -uroot -proot -hmysql-57.default.svc.tcce.local -e "show databases;"
```

## 外部通过暴露的端口连接测试
![成功](success.png)
