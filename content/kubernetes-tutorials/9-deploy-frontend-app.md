---
title: "9. 部署前端应用"
date: 2019-04-03T20:13:24+08:00
draft: false
categories:
  - Kubernetes
tags:
  - kubernetes
  - vuejs
  - kubernetes-tutorials-1
---

<!--more-->

## 约定
Kubernetes 命名空间：platform-test

## 准备 gitlab ci 文件
```yaml
#.gitlab-ci.yml

cache:
  paths:
    - dist/
    - node_modules/

stages:
  - staging
  - production

staging:
  tags:
    - staging
  stage: staging
  script:
    - npm install
    - npm run build
    - make staging
  except:
    - master # 除了 Master 分支之外，都进行 staging 环境的部署

production:
  tags:
    - production
  stage: production
  script:
    - npm install
    - npm run build
    - make push_production
  only:
    - master
```

```makefile
# Makefile
LOCAL_PREFIX := platform.tcce.pro
STAGING_PREFIX := harbor.tcce.local/tcce-pro
PRODUCTION_PREFIX := xxxx.aliyuncs.com/tcce-pro
NAME := platform-front
SNAPSHOT_VERSION:=$(shell date +%Y%m%d%H%M%S)
PRODUCTION_VERSION := $(shell node -p "require('./package.json').version")

.PHONY: docker push_staging push_production

docker:
	docker image prune -f
	docker build . -t ${LOCAL_PREFIX}/${NAME}:latest

push_staging: docker
	docker tag ${LOCAL_PREFIX}/${NAME}:latest ${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}
	docker push ${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}

staging: push_staging
	kubectl apply -f ./deploy/staging/
	kubectl -n platform-test set image deploy/platform-front platform-front=${STAGING_PREFIX}/${NAME}:${SNAPSHOT_VERSION}

push_production: docker
	docker tag ${LOCAL_PREFIX}/${NAME}:latest ${PRODUCTION_PREFIX}/${NAME}:${PRODUCTION_VERSION}
	docker push ${PRODUCTION_PREFIX}/${NAME}:${PRODUCTION_VERSION}
```

## K8s 部署文件

### 测试环境

文件位于项目的 deploy/staging 目录下：

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
  namespace: platform-test
  name: platform-front
  labels:
    app: platform-front
spec:
  revisionHistoryLimit: 5
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: platform-front
    spec:
      containers:
        - image: harbor.tcce.local/tcce-pro/platform-front:latest
          imagePullPolicy: Always
          name: platform-front
          ports:
            - containerPort: 80
              name: platform-front
      imagePullSecrets:
        - name: staging
  selector:
    matchLabels:
      app: platform-front
```

```yaml
# 3-service.yml
apiVersion: v1
kind: Service
metadata:
  name: platform-front
  namespace: platform-test
  labels:
    app: platform-front
spec:
  ports:
    - name: platform-front-http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: platform-front
  sessionAffinity: None
  type: ClusterIP
```

```yaml
# 4-ingress.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: platform-front-ingress
  namespace: platform-test
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: /
            backend:
              serviceName: platform-front
              servicePort: 80
```
