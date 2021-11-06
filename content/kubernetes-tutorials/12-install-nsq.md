---
title: "12. 安装 NSQ"
date: 2019-04-08T20:22:24+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - nsq
  - kubernetes-tutorials-1
---

<!--more-->

## 准备配置文件
```yaml {hl_lines=[48,"72-77","129-134",194]}
apiVersion: v1
kind: Namespace
metadata:
  name: nsq

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: nsq
  name: nsqlookupd
  labels:
    app: nsq
    component: nsqlookupd
spec:
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: nsq
        component: nsqlookupd
    spec:
      containers:
        - image: nsqio/nsq:v1.1.0
          name: nsqlookupd
          command:
            - /nsqlookupd
          ports:
            - containerPort: 4160
              name: tcp
            - containerPort: 4161
              name: http
          livenessProbe:
            httpGet:
              port: 4161
              path: /ping
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              port: 4161
              path: /ping
            initialDelaySeconds: 10
  selector:
    matchLabels:
      app: nsq
      component: nsqlookupd
  serviceName: nsqlookupd
  replicas: 3 # (1)

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: nsq
  name: nsqd
  labels:
    app: nsq
    component: nsqd
spec:
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: nsq
        component: nsqd
    spec:
      containers:
        - image: nsqio/nsq:v1.1.0
          name: nsqd
          command:
            - /nsqd
            - -lookupd-tcp-address # (2)
            - nsqlookupd-0.nsqlookupd.nsq.svc.tcce.local:4160
            - -lookupd-tcp-address
            - nsqlookupd-1.nsqlookupd.nsq.svc.tcce.local:4160
            - -lookupd-tcp-address
            - nsqlookupd-2.nsqlookupd.nsq.svc.tcce.local:4160
            - -broadcast-address
            - $(POD_IP)
          ports:
            - containerPort: 4150
              name: tcp
            - containerPort: 4151
              name: http
          livenessProbe:
            httpGet:
              port: 4151
              path: /ping
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              port: 4151
              path: /ping
            initialDelaySeconds: 10
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
  selector:
    matchLabels:
      app: nsq
      component: nsqd
  serviceName: nsqd
  replicas: 3

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: nsq
  name: nsqadmin
  labels:
    app: nsq
    component: nsqadmin
spec:
  revisionHistoryLimit: 2
  template:
    metadata:
      labels:
        app: nsq
        component: nsqadmin
    spec:
      containers:
        - name: nsqadmin
          image: nsqio/nsq:v1.1.0
          command:
            - /nsqadmin
            - -lookupd-http-address #(3)
            - nsqlookupd-0.nsqlookupd.nsq.svc.tcce.local:4161
            - -lookupd-http-address
            - nsqlookupd-1.nsqlookupd.nsq.svc.tcce.local:4161
            - -lookupd-http-address
            - nsqlookupd-2.nsqlookupd.nsq.svc.tcce.local:4161
          ports:
            - containerPort: 4171
              name: http
          livenessProbe:
            httpGet:
              port: 4171
              path: /ping
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              port: 4171
              path: /ping
            initialDelaySeconds: 10
  selector:
    matchLabels:
      app: nsq
      component: nsqadmin
  replicas: 1
---
apiVersion: v1
kind: Service
metadata:
  name: nsqadmin
  namespace: nsq
  labels:
    app: nsq
    component: nsqadmin
spec:
  ports:
    - port: 4171
      name: http
      targetPort: 4171
      nodePort: 4171
  selector:
    app: nsq
    component: nsqadmin
  sessionAffinity: None
  type: LoadBalancer

---
apiVersion: v1
kind: Service
metadata:
  name: nsqlookupd
  namespace: nsq
  labels:
    app: nsq
    component: nsqlookupd
spec:
  ports:
    - port: 4160
      name: tcp
      targetPort: 4160
    - port: 4161
      name: http
      targetPort: 4161
  selector:
    app: nsq
    component: nsqlookupd
  clusterIP: None # (4)

---
apiVersion: v1
kind: Service
metadata:
  name: nsqd
  namespace: nsq
  labels:
    app: nsq
    component: nsqd
spec:
  ports:
    - port: 4150
      name: tcp
      targetPort: 4150
    - port: 4151
      name: http
      targetPort: 4151
  selector:
    app: nsq
    component: nsqd
  type: ClusterIP
```

重点关注：

- (1) ：部署 `3` 个 `StatefulSet` 类型的 `nsqlookupd` 实例，这里的数量是 [NSQ 官方文档中推荐的](https://nsq.io/deployment/production.html#nsqlookupd)；
- (2) ：为 `nsq` 分别设置多个 `nsqlookupd` 的地址，因为各个 `nsqlookupd` 相互没有任何关系；
- (3) ：为 `nsqadmin` 分别设置多个 `nsqlookupd` 的地址；
- (4) ：设置 `nsqlookupd service`  的 `clusterIP` 为 `None`，使其成为 `Headless Service`，此时，`service` 对应的每个 `Pod` 的 DNS 为 `nslookupd-{0..N}.nslookupd.nsq.svc.tcce.local`，详见[文档](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id)，由此支持 (2) (3) 的配置。

## 验证

访问 NSQAdmin http://10.172.23.166:4171

![NSQ Admin](nsqadmin.png)
