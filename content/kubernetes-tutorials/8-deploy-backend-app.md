---
title: "8. 部署 Java 后端应用"
date: 2019-04-03T19:06:35+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - spring-boot
  - kubernetes-tutorials-1
---

<!--more-->

## 约定

- Kubernetes 命名空间：platform-test
- 配置文件名称：platform-back-config

## 准备 ConfigMap

首先需要将后端 Spring 应用的配置文件写入 Kubernetes ConfigMap：

```shell
kubectl -n platform-test create configmap platform-back-config --from-file application.yml
```

查看写入结果：
```shell
kubectl -n platform-test get configmap platform-back-config -o yaml

apiVersion: v1
data:
  application.yml: |+
    server:
      port: 9998
      address: 0.0.0.0
      servlet:
        context-path: /api
    spring:
      application:
        name: platform-back
      datasource:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://mysql-57.default.svc.tcce.local:3306/platform
        username: root
        password: root
      cloud:
        consul:
          host: consul.default.svc.tcce.local
          port: 8500
          discovery:
            prefer-ip-address: true
...

kind: ConfigMap
metadata:
  creationTimestamp: "2019-03-29T03:33:41Z"
  name: platform-back-config
  namespace: platform-test
  resourceVersion: "313206"
  selfLink: /api/v1/namespaces/platform-test/configmaps/platform-back-config
  uid: 72b02c04-51d3-11e9-a1eb-1c1b0dfdf442
```

## 准备拉取镜像的 Secret

Kubernetes 在拉取私有镜像时，需要相关认证信息，所以创建 docker-registry 类型的 secret 来进行保存：

```shell
kubectl create secret docker-registry staging --docker-server=https://harbor.tcce.local --docker-username=<USERNAME> --docker-password=<PASSWORD>
```
查看创建的 docker-registry secret:

```shell
$kubectl describe secret staging

Name:         staging
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  161 bytes
```

## 准备 gitlab-ci.yml 文件
使用 gitlab ci 来进行应用的镜像打包、推送及部署：

```yaml
# .gitlab-ci.yml
cache:
  paths:
    - target/*.jar

stages:
  - test
  - staging
  - production

test:
  tags:
    - staging
  stage: test
  script:
    - mvn clean test -Dtest="*,!ApiTest" -DfailIfNoTests=false

staging:
  tags:
    - staging
  stage: staging
  script:
    - mvn clean package -DskipTests
    - make staging
  dependencies:
    - test
  except:
    - master

production:
  tags:
    - production
  stage: production
  script:
    - mvn clean package -DskipTests
    - make docker
    - make push_production
    - kubectl apply -f ./deploy/production/
  only:
    - master
```

```makefile
# Makefile
LOCAL_PREFIX := platform.tcce.pro
STAGING_PREFIX := harbor.tcce.local/tcce-pro # 私有 Docker Registry
PRODUCTION_PREFIX := xxxx.aliyuncs.com/tcce-pro
NAME := platform-back
SNAPSHOT_VERSION:=$(shell date +%Y%m%d%H%M%S)
PRODUCTION_VERSION := $(shell mvn -q -Dexec.executable="echo" -Dexec.args='$${project.version}' --non-recursive exec:exec)

.PHONY: docker push_staging push_production

docker:
	docker image prune -f
	docker build . -t ${LOCAL_PREFIX}/${NAME}:latest

push_staging: docker
	docker tag ${LOCAL_PREFIX}/${NAME}:latest ${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}
	docker push ${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}

staging: push_staging
	kubectl apply -f ./deploy/staging/
	# 滚动更新
	kubectl -n platform-test set image deploy/platform-back platform-back=${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}

push_production: docker
	docker tag ${LOCAL_PREFIX}/${NAME}:latest ${PRODUCTION_PREFIX}/${NAME}:${PRODUCTION_VERSION}
	docker push ${PRODUCTION_PREFIX}/${NAME}:${PRODUCTION_VERSION}
```

```dockerfile
FROM azul/zulu-openjdk-alpine:8
VOLUME /platform/back/logs
VOLUME /platform/back/config

ADD target/platform.jar /platform/back/app.jar

EXPOSE 9998

WORKDIR /platform/back
```

## k8s 部署文件

### 测试环境

测试环境的部署文件位于项目 deploy/staging 目录中：

```yaml
# 1-namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: platform-test
```

```yaml
# 2-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: platform-test # 注意命名空间
  name: platform-back
  labels:
    app: platform-back
spec:
  strategy:
    type: Recreate # 每次部署都是重新创建
  template:
    metadata:
      labels:
        app: platform-back
    spec:
      containers:
        - image: harbor.tcce.local/tcce-pro/platform-back:latest
          name: platform-back
          command:
            - java
            - -jar
            - /platform/back/app.jar
            - --spring.profiles.active=production
            - --spring.config.location=/platform/back/conf/application.yml
          ports:
            - containerPort: 9998 # 暴露的端口要与配置文件中的一致
              name: platform-back
          volumeMounts: # 将 ConfigMap 中的配置文件挂载到对应目录
            - mountPath: /platform/back/conf/
              name: platform-back-config
              readOnly: true
      imagePullSecrets: # 设置拉取镜像的 Secret
        - name: staging
      volumes:
        - name: platform-back-config # 挂载 ConfigMap 卷
          configMap:
            name: platform-back-config
  selector:
    matchLabels:
      app: platform-back
```
```yaml
# 3-service.yml
apiVersion: v1
kind: Service
metadata:
  name: platform-back
  namespace: platform-test
  labels:
    app: platform-back
spec:
  ports:
    - name: platform-back-http
      port: 9998
      protocol: TCP
      targetPort: 9998
  selector:
    app: platform-back # 选择所有拥有 app: platform-back 标签的 Pod
  sessionAffinity: None
  type: ClusterIP
```

## 使用 Ingress 将服务暴露至集群外

```yaml
# 4-ingress.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: platform-back-ingress
  namespace: platform-test
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /api
        backend:
          serviceName: platform-back
          servicePort: 9998
```

## 验证

待所有 Pod 启动成功后，浏览器访问：


http://10.172.23.166/api/swagger-ui.html
