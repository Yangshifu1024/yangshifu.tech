---
title: "7. 部署 Nginx Ingress"
date: 2019-04-02T22:41:13+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - ingress
  - nginx
  - kubernetes-tutorials-1
---

<!--more-->

## 修改 api-server 配置
默认情况下 Kubernetes 的 NodePort 分配范围在 30000-32767 之间，这里修改为 1-62375：

```shell
$ sudo vim /etc/kubernetes/manifest/kube-apiserver.yaml
```

修改：
```yaml
- --service-node-port-range=1-65535
```

## 必要配置

部署默认 HTTP 后端：

```yaml
# default-http-backend.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  namespace: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      containers:
        - name: http-svc
          image: gcr.azk8s.cn/google_containers/echoserver:1.8
          ports:
            - containerPort: 8080
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP

---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: ingress-nginx
  labels:
    app: default-http-backend
spec:
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: default-http-backend
```

这里部署了一个默认 HTTP 后端，在前端 INGRESS 没有找到对应的后端时，默认使用这个 Pod 来进行响应。


下载 nginx ingress 配置：

```shell
$ curl -o nginx-ingress-controller.yml https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```
修改如下：

```yaml
# 新增 spec.affinity，将 nginx-ingress 安装到 test1 节点
affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - test1.tcce.local
```
```shell
kubectl apply -f nginx-ingress-controller.yml
```

## Service

```shell
$ curl -o nginx-ingress-service.yml https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

修改如下：
```yaml
# 新增 spec.ports.80.nodePort 和 spec.ports.443.nodePort
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 80 # 1
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
      nodePort: 443 # 2
```

```shell
kubectl apply -f nginx-ingress-service.yml
```

## 测试

因为目前没有部署其他 Ingress 后端，所以所有对 Ingress 的请求都将被重定向到 default-http-backend：

![Echo-Server](echo-server.png)

## 将 Kubernetes Dashboard 暴露

在系列文章的第一篇中，安装 Dashboard 时修改了 `spec.type` 字段，现在需要修改回去：

```shell
$ kubectl edit svc/kubernetes-dashboard -n kube-system
```

将 `spec.type` 改回：`ClusterIP`，保存生效。

因为最新版本的 Dashboard 已经禁止 HTTP 环境登录，所以需要自签名证书来实现 HTTPS：

```shell
# 生成证书
$ mkdir tls
$ openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout tls/dashboard.key -out tls/dashboard.crt -subj "/CN=dashboard.tcce.local"
```

创建 TLS 类型的 Secret 到 kube-system 命名空间下供 Ingress 使用：

```shell
$ kubectl -n kube-system create secret tls secret-ca-dashboard --key tls/dashboard.key --cert tls/dashboard.crt
```

```yaml
# kubernetes-dashboard-ingress.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - secretName: secret-ca-dashboard
  rules:
  - host: dashboard.tcce.local
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

```shell
kubectl apply -f kubernetes-dashboard-ingress.yml
```

![Kubernetes Dashboard](dashboard.png)
