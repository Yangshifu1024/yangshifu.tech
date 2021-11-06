---
title: "0. Kubernetes 基础"
date: 2019-04-02T19:40:55+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - kubernetes-tutorials-1
---

<!--more-->

## 1.1 概念与术语

### Pod

Pod 是 Kubernetes 的最小调度单元，同一 Pod 中的容器共享网络名称空间和存储资源，这些容器可以经由本地回环口 lo 直接通信（127.0.0.1 或 localhost），但彼此之间又在 Mount、User 及 PID 等 Namespace 上保持了隔离。通常只应包含一个主容器以及必要的辅助型容器（sidecar）。

### Label

资源标签是将资源进行分类的标识符，其实就是一个键值型数据。

### Label Selector

标签选择器是一种根据 Label 来过滤符合条件的资源对象的机制。

### Pod Controller

用户通常不会直接部署及管理 Pod 对象，如 `kubectl run`，而是借助于控制器。

控制器是一种管理 Pod 生命周期的资源抽象，它们是一类对象，而非单个资源对象，包括：

- ReplicationController
- ReplicaSet
- Deployment
- StatefulSet
- Job
- ……

### Service

Service 是建立在一组 Pod 对象之上的资源抽象，它通过标签选择器选定一组 Pod 对象，并为这组 Pod 对象定义一个统一的固定访问入口（通常是一个 IP 地址）；若 Kubernetes 集群存在 DNS 附件（KubeDNS），它就会在 Service 创建时为其自动配置一个 DNS 名称以便客户端进行服务发现。

到达 Service IP 的请求将会被负载均衡至其后的端点——各个 Pod 对象之上，因此 Service 从本质上来讲是一个四层代理服务。

### Volume

存储卷是独立于容器文件系统之外的存储空间，常用于扩展容器的存储空间并为它提供持久存储能力。

在 Kubernetes 中大体可分为临时卷、本地卷和网络卷。

### Name 和 Namespace

Name 是 Kubernetes 集群中资源对象的标识符，它们的作用域通常是名称空间 Namespace，在同一个名称空间中，同一类型资源对象的名称必须具有**唯一性**。

名称空间通常用于实现租户或项目的资源隔离，从而形成逻辑分组。

在创建资源时未指定 Namespace 时，它们都属于默认的名称空间：default。

### Annotation

注解是另一种附加在对象之上的键值类型的数据，但它们拥有更大的数据容量。Annotation 常用于将各种**非标识型元数据（metadata）**附加到对象上，但它不能用于标识和选择对象。

### Ingress

Kubernetes 将 Pod 对象和外部网络环境进行了隔离，Pod 和 Service 等对象间的通信都使用其内部专用地址进行，如若需要开放某些 Pod 对象给外部，通常使用 Service 和 Ingress。

> Service 暴露至外部通常使用 NodePort

## 2. 组件
一个典型的 Kubernetes 集群由多个工作节点 （Worker Node）和一个集群控制平面（Control Plane，即 Master），以及一个集群状态存储系统（etcd）组成。

### 2.1 Master 组件

#### API Server

API Server 负责提供 RESTful 风格的 Kubernetes API，负责接收、校验并相应所有的 REST 请求，并发往集群，结果状态被持久存储于 etcd 中。因此，API Server 是整个集群的网关。

> `kubectl get pod -n kube-system | grep apiserver`

#### 集群状态存储

Kubernetes 集群的所有状态信息都持久存储于 etcd 中，etcd 不仅能够存储键值数据，而且还提供了监听（watch）机制，用于监听和推送变更。Kubernetes 集群系统中，etcd 中的键值发生变化时会通知到 API Server，并由其通过 watch API 向客户端输出，实现了集群各组件的高效协同。

#### Controller Manager

Kubernetes 中，集群级别的大多数功能都是由几个被成为控制器的进程执行实现的。

- 生命周期管理：包括 Namespace 创建和生命周期、Event 垃圾回收、Pod 终止相关的垃圾回收、级联垃圾回收及 Node 垃圾回收等；
- API 业务逻辑：例如，由 ReplicaSet 执行的 Pod 扩展等。

> `kubectl get pod -n kube-system | grep controller-manager`

#### Scheduler

API Server 确认 Pod 对象的创建请求之后，需要由 Scheduler 根据集群**各节点的可用资源状态，以及容器的资源需求**作出调度决策。

### 2.2 Node 组件

#### Kubelet

Kubelet 是运行于工作节点之上的守护进程，它从 API Server 接收关于 Pod 对象的配置信息并确保它们处于期望的状态（Desired state）。Kubelet 会在 API Server 上注册当前节点，定期向 Master 汇报节点资源使用情况，并通过 cAdvisor 监控容器和节点的资源使用情况。

> `ps auxww | grep kubelet`

#### Container Runtime

Container Runtime 负责下载镜像并运行容器，常见为 Docker。

#### Kube Proxy

每个工作节点上都需要运行一个 kube-proxy 守护进程，它能够按需为 Service 资源对象生成 iptables 或 ipvs 规则，从而捕获访问当前 Service 的 ClusterIP 的流量并将其转发至正确的后端 Pod 对象。

## 3. 网络模型

Kubernetes 网络通信主要存在四种类型：

- 同一 Pod 内的容器间通信
- Pod 之间的通信
- Pod 与 Service 间的通信
- 集群外部与 Service 间的通信

Pod 网络由 Kubernetes 的网络插件配置实现，具体使用的网络地址可在管理配置网络插件时指定，如 10.244.0.0/16 网络；而 Service 的网络则由 Kubernetes 集群予以指定，Service 的 IP 地址是集群提供服务的接口，也成为 Cluster IP，如：10.96.0.0/12 网络。

Kubernetes 集群至少包含三个网络：

- 各主机（Master、Node 和 etcd 等）自身所属的网络，需要管理员在集群构建之前确定；
- Pod 资源对象的网络，它是一个虚拟网络，需要借助 kubenet 插件或 CNI 插件实现；
- Service 资源对象的网络，通过 Node 之上的 kube-proxy 配置为 iptables 或 ipvs 规则，进行流量转发。

## 4. 资源配置清单

### Namespace

> `kubectl create ns helloworld --dry-run -o yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: default
```

对于创建 Namespace 来说，仅需提供 `metadata.name` 属性即可。

### Deployment

> `kubectl create deployment helloworld --image=helloworld:latest --dry-run -o yaml`


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <string>
spec:
  replicas: <integer>
  selector:
    matchLabels:
      <key>:<value>
  template:
    metadata:
      labels:
        <key>:<value>
    spec:
      containers:
      - name: <string>
        image: <string>
        command: [<optional-string>]
        args: [<optional-string>, <optional-string>]
        ports:
        - containerPort: <integer>
          name: <string>
```
