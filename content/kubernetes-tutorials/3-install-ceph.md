---
title: "3. 安装 Ceph 存储集群"
date: 2019-04-02T22:11:56+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - ceph
  - kubernetes-tutorials-1
---

<!--more-->

## 安装 Ceph Operator

执行
```
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml
```

查看 Operator 状态：

```shell
$ kubectl get pod -n rook-ceph-system -o wide
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-agent-g288g                 1/1     Running   0          66m
rook-ceph-agent-pk8sf                 1/1     Running   0          66m
rook-ceph-agent-vxvlf                 1/1     Running   0          66m
rook-ceph-operator-769d56c4b5-r7fqs   1/1     Running   0          66m
rook-discover-6ljlp                   1/1     Running   0          66m
rook-discover-8vwmk                   1/1     Running   0          66m
rook-discover-dtrvf                   1/1     Running   0          66m
```

等待所有 Pod 成功，如上输出所示。

## 安装 Ceph Cluster

```shell
$ curl -o ceph-cluster.yml https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```

```yaml
# ceph-cluster.yaml
# ... 省略
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v13
    allowUnsupported: false
  dataDirHostPath: /opt/rook
  mon:
    count: 3
    allowMultiplePerNode: true
  dashboard:
    enabled: true
  storage:
    useAllNodes: true
    useAllDevices: false
```

修改其中的 `CephCluster.spec.dataDirHostPath`，保存并执行：`kubectl apply -f ceph-cluster.yml`

查看 Pod 准备完毕：

```shell
$ kubectl -n rook-ceph get pod
NAME                                           READY   STATUS      RESTARTS   AGE
rook-ceph-mgr-a-67d98c95c9-zrrvx               1/1     Running     0          60m
rook-ceph-mon-a-74d58ffccf-qjsxw               1/1     Running     0          61m
rook-ceph-mon-b-5cd778d786-vjttx               1/1     Running     0          60m
rook-ceph-mon-c-784698d9dc-nx5mk               1/1     Running     0          60m
rook-ceph-osd-0-587d564cc6-bdj2q               1/1     Running     0          59m
rook-ceph-osd-1-fffc695ff-nsrps                1/1     Running     0          59m
rook-ceph-osd-2-569c97878-54x74                1/1     Running     0          59m
rook-ceph-osd-prepare-test1.tcce.local-txdmp   0/2     Completed   0          60m
rook-ceph-osd-prepare-test2.tcce.local-q5v4h   0/2     Completed   0          60m
rook-ceph-osd-prepare-test3.tcce.local-57rv7   0/2     Completed   0          60m
```
直到如上输出，集群准备完毕。

## 配置 StorageClass

```shell
$ curl -o ceph-storageclass.yml https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/storageclass.yaml
```

为了保障数据不被删除，修改为：
```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  clusterNamespace: rook-ceph
  fstype: xfs
reclaimPolicy: Retain #新增这一行
```

## 测试

准备一个 PVC 测试文件如下：
```yaml
# pvc-test.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```

```shell
$ kubectl apply -f pvc-test.yml
```

检查是否成功：
```shell
$ kubectl describe pvc/pvc-test
Name:          pvc-test
Namespace:     default
StorageClass:  rook-ceph-block
Status:        Bound
Volume:        pvc-d076476a-5137-11e9-bdd7-1c1b0dfdf442
Labels:        <none>
Annotations:   ...
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Events:
  Type       Reason                 Age                From                                                                                         Message
  ----       ------                 ----               ----                                                                                         -------
  Normal     Provisioning           35s                ceph.rook.io/block_rook-ceph-operator-769d56c4b5-r7fqs_d5b0955b-512c-11e9-8139-f270a0a59ec0  External provisioner is provisioning volume for claim "default/pvc-test"
  Normal     ProvisioningSucceeded  35s                ceph.rook.io/block_rook-ceph-operator-769d56c4b5-r7fqs_d5b0955b-512c-11e9-8139-f270a0a59ec0  Successfully provisioned volume pvc-...
  Normal     ExternalProvisioning   34s (x2 over 34s)  persistentvolume-controller                                                                  waiting for a volume to be created, either by external provisioner "ceph.rook.io/block" ...
Mounted By:  <none>
```

清理测试：`kubectl delete -f pvc-test.yml`
