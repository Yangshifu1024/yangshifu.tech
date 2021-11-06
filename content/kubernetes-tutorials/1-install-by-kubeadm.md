---
title: "1. 使用 Kubeadm 安装 Kubernetes 集群"
date: 2019-04-02T20:50:59+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - kubeadm
  - kubernetes-tutorials-1
---

<!--more-->

## 服务器准备

系统版本：CentOS 7

|     Hostname     |      IP       |  角色  |
| :--------------: | :-----------: | :----: |
| test1.tcce.local | 10.172.23.166 | Master |
| test2.tcce.local | 10.172.23.167 | Worker |
| test3.tcce.local | 10.172.23.168 | Worker |

## 系统配置

### 设置主机名
分别配置每台服务器：

```shell
# test1.tcce.local
hostnamectl set-hostname test1.tcce.local
```

```shell
# test2.tcce.local
$ hostnamectl set-hostname test2.tcce.local
```

```shell
# test3.tcce.local
$ hostnamectl set-hostname test3.tcce.local
```

### 配置 hosts

每台服务器都配置
```python
# /etc/hosts
10.172.23.166 test1.tcce.local
10.172.23.167 test2.tcce.local
10.172.23.168 test3.tcce.local
```

### 禁止 selinux
```shell
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 禁用 swap
```shell
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 优化内核参数
```shell
$ cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

$ sudo cp kubernetes.conf  /etc/sysctl.d/kubernetes.conf

$ sudo sysctl -p /etc/sysctl.d/kubernetes.conf
```

### 配置 Epel 源

```shell
$ sudo yum install -y https://mirrors.huaweicloud.com/epel/epel-release-latest-7.noarch.rpm

$ sudo sed -i "s/#baseurl/baseurl/g" /etc/yum.repos.d/epel.repo
$ sudo sed -i "s/mirrorlist/#mirrorlist/g" /etc/yum.repos.d/epel.repo
$ sudo sed -i "s@http://download.fedoraproject.org/pub@https://mirrors.huaweicloud.com@g" /etc/yum.repos.d/epel.repo
```

### 禁用 fastmirror 插件

```shell
$ sudo sed -i 's/^enabled=.*/enabled=0/' /etc/yum/pluginconf.d/fastestmirror.conf
```

### 安装 Docker

移除旧版本：
```shell
$ sudo yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine
```

安装依赖包：
```shell
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

配置 docker 仓库：
```shell
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

安装：
```shell
$ sudo yum install docker-ce
```

配置开机启动：
```shell
$ sudo systemctl enable docker
```

### 安装 kubeadm

```shell
$ cat > kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

$ sudo mv kubernetes.repo /etc/yum.repos.d/
$ sudo yum install -y kubelet kubeadm kubectl
$ sudo systemctl --now enable kubelet
```

### 更新系统

```shell
$ sudo yum update
```

配置结束后重启服务器。

## 基础镜像
注意：因为谷歌官方镜像被墙，无法直接初始化。

Azure 中国提供了相关镜像的代理服务，详见[链接](http://mirror.azure.cn/help/gcr-proxy-cache.html)。

```shell
# pull.sh
docker pull gcr.azk8s.cn/google_containers/kube-apiserver:v1.14.0
docker pull gcr.azk8s.cn/google_containers/kube-controller-manager:v1.14.0
docker pull gcr.azk8s.cn/google_containers/kube-scheduler:v1.14.0
docker pull gcr.azk8s.cn/google_containers/kube-proxy:v1.14.0
docker pull gcr.azk8s.cn/google_containers/pause:3.1
docker pull gcr.azk8s.cn/google_containers/etcd:3.3.10
docker pull gcr.azk8s.cn/google_containers/coredns:1.3.1
docker pull gcr.azk8s.cn/google_containers/kubernetes-dashboard-amd64:v1.10.1
docker pull gcr.azk8s.cn/kubernetes-helm/tiller:v2.13.1

docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.23.0

docker tag gcr.azk8s.cn/google_containers/kube-apiserver:v1.14.0 k8s.gcr.io/kube-apiserver:v1.14.0
docker tag gcr.azk8s.cn/google_containers/kube-controller-manager:v1.14.0 k8s.gcr.io/kube-controller-manager:v1.14.0
docker tag gcr.azk8s.cn/google_containers/kube-scheduler:v1.14.0 k8s.gcr.io/kube-scheduler:v1.14.0
docker tag gcr.azk8s.cn/google_containers/kube-proxy:v1.14.0 k8s.gcr.io/kube-proxy:v1.14.0
docker tag gcr.azk8s.cn/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag gcr.azk8s.cn/google_containers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag gcr.azk8s.cn/google_containers/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker tag gcr.azk8s.cn/google_containers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
docker tag gcr.azk8s.cn/kubernetes-helm/tiller:v2.13.1 gcr.io/kubernetes-helm/tiller:v2.13.1
```

## Master 和 Nodes

### Master

#### 初始化：
```shell
$ sudo kubeadm init --kubernetes-version=v1.14.0 --apiserver-advertise-address 10.172.23.166 --pod-network-cidr 10.244.0.0/16 --service-dns-domain tcce.local
```
其中 166 为本机对外广播的IP地址。

命令将输出：

```shell
# 运行以下输出的命令
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
保存此命令，将用于 Worker 节点。
```shell
$ kubeadm join 10.172.23.166:6443 --token qomkll.6q9z020r00kycao6 --discovery-token-ca-cert-hash sha256:6c896c8056efd1882aad8669e1144748116f44377e81e1843df7d6df2510e9d7
```

此时运行 `kubectl get node`，将输出以下信息：

```shell
$ kubectl get nodes
NAME          STATUS     ROLES    AGE     VERSION
tcce-test-1   NotReady   master   4m42s   v1.14.0
```

可以看到状态为 NotReady。

#### Pod 网络

参考：[pod-network](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network) 中的 `flannel` 部分。

```shell
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

稍等几秒，再次 `kubectl get nodes` 查看：
```shell
$ kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
tcce-test-1   Ready    master   8m15s   v1.14.0
```
master 节点状态为 Ready。

默认情况下，k8s 将不会使用 Master 节点来部署 Pods，但为了充分利用资源，这里我们将取消这个限制：

```shell
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Node

在其余两台服务器执行以下命令来加入集群：

```shell
$ sudo kubeadm join 10.172.23.166:6443 --token qomkll.6q9z020r00kycao6 --discovery-token-ca-cert-hash sha256:6c896c8056efd1882aad8669e1144748116f44377e81e1843df7d6df2510e9d7
```

其中的 `token` 和 `discovery-token-ca-cert-hash` 可以通过如下方式在 Master 上获取：

```shell
# token
$ kubeadm token list
```

```shell
# discovery-token-ca-cert-hash
$ openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1
```

也可以重新创建 `token`：
```shell
$ kubeadm token create --print-join-command
```
此命令会在创建 token 的同时输出其他 Node 加入时直接可用的命令。

至此，整个集群已经成功搭建：

```shell
$ kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
tcce-test-1   Ready    master   11m   v1.14.0
tcce-test-2   Ready    <none>   85s   v1.14.0
tcce-test-3   Ready    <none>   11s   v1.14.0
```

## 安装控制台

参考：[`dashboard-releases`](https://github.com/kubernetes/dashboard/releases)。

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

执行 `kubectl get pods --namespace=kube-system` 来查看控制台的运行结果：

```shell
$ kubectl get pods --namespace=kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-82v5f               1/1     Running   0          15m
coredns-86c58d9df4-l27m4               1/1     Running   0          15m
etcd-tcce-test-1                       1/1     Running   0          14m
kube-apiserver-tcce-test-1             1/1     Running   0          14m
kube-controller-manager-tcce-test-1    1/1     Running   0          14m
kube-flannel-ds-amd64-hsbsz            1/1     Running   0          6m9s
kube-flannel-ds-amd64-jfgcc            1/1     Running   0          7m50s
kube-flannel-ds-amd64-t5kcj            1/1     Running   0          4m55s
kube-proxy-7r4rh                       1/1     Running   0          6m9s
kube-proxy-c7kwt                       1/1     Running   0          15m
kube-proxy-tkshg                       1/1     Running   0          4m55s
kube-scheduler-tcce-test-1             1/1     Running   0          14m
kubernetes-dashboard-57df4db6b-xjzh9   1/1     Running   0          17s
```

执行 `kubectl get service --namespace=kube-system` 来查看控制台监听信息：
```shell
$ kubectl get service --namespace=kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   17m
kubernetes-dashboard   ClusterIP   10.111.88.22   <none>        443/TCP         90s
```

执行 `kubectl proxy -p 8000 --address 10.172.23.166` 来开启访问，访问 `http://10.172.23.166:8000`，提示 `Forbidden`。

编辑 dashboard 配置文件：
```shell
$ kubectl -n kube-system edit service kubernetes-dashboard
```

将 `type: ClusterIP` 修改为：`type: NodePort`，保存后自动生效。

再次查看 service 信息，`kubectl -n kube-system get service`：
```shell
$ kubectl -n kube-system get service
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP   80m
kubernetes-dashboard   NodePort    10.96.122.12   <none>        443:30700/TCP   7m57s
```
可以看到 dashboard 的 443 端口映射到了 30700 端口，在浏览器中打开：`https://10.172.23.166:30700`，进入登录页面，这里使用 Token 登录：
{% asset_img login-form.png %}
![登录页面](login-form.png)

首先获取 secret 的 key：

```shell
$ kubectl -n kube-system get secret | grep dashboard-token
```
```shell
$ kubectl -n kube-system get secret | grep dashboard-token
kubernetes-dashboard-token-sz5qm                 kubernetes.io/service-account-token   3      11m
```

复制 `kubernetes-dashboard-token-sz5qm`，执行：

```shell
$ kubectl -n kube-system describe secret kubernetes-dashboard-token-sz5qm
```

```shell
$ kubectl -n kube-system describe secret kubernetes-dashboard-token-sz5qm
Name:         kubernetes-dashboard-token-sz5qm
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: 6ca3cbf2-4d34-11e9-9c13-1c1b0dfdf442

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6I...

```

复制最下面的 token 段，粘贴并登录：
![权限警告](permission-warnings.png)
可以看到，此时虽然登录了，但是依然有报错，出错是因为 service dashboard 没有对应的权限，所以接下来为 dashboard 配置对应的权限：
创建文件 `dashboard-admin.yml`：

```yaml
# dashboard-admin.yml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

执行命令：
```shell
$ kubectl create -f dashboard-admin.yml
```
退出登录或刷新 dashboard：
![成功](success.png)

至此，k8s 集群基本搭建完成。

## 拆卸集群
**警告，以下操作不是必须步骤，仅在需要重置整个集群时使用。**

处理各个节点：
```shell
$ kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
$ kubectl delete node <node name>
```

重置：
```shell
$ kubeadm reset
```
