# Kubernetes部署
该文档提供了在kubernetes上运行micro的指南。

### 依存关系
   在kubernetes上，我们建议运行[etcd](https://github.com/etcd-io/etcd)和[nats](https://github.com/nats-io/nats-server)。
        - Etcd用于高度可扩展的服务发现
        - NATS用于异步消息传递
#### 要安装etcd（[说明](https://github.com/helm/charts/tree/master/stable/etcd-operator)）
```
# 注意命名空间 【会在后面微服务利用用到】
helm install --namespace ${namespace} --name ${appName} --set customResources.createEtcdClusterCRD=true stable/etcd-operator

# to later uninstall etcd
helm delete ${appName}

# 例:
helm install --namespace srv --name etcd-operator --set customResources.createEtcdClusterCRD=true stable/etcd-operator

# to later uninstall etcd
helm delete etcd-operator

```
```
# rancher 可以去应用商品安装
# rancher 流水线 代码
stages:
- name: Etcd
  steps:
  - applyAppConfig:
      catalogTemplate: cattle-global-data:library-etcd-operator
      version: 0.9.0
      answers:
        customResources.createEtcdClusterCRD: "true"
      name: etcd-operator
      targetNamespace: srv
```
#### 安装nat
（[nats](https://github.com/nats-io/nats-server)）
```
# 暂无安装后续完善
```

### 部署服务
这是微服务的k8s部署示例
```
# etcd-cluster-client.srv 注意: srv 为 etcd 服务所在命名空间
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: default
  name: greeter
spec:
  replicas: 1
  selector:
    matchLabels:
      name: greeter-srv
      micro: service
  template:
    metadata:
      labels:
        name: greeter-srv
        micro: service
    spec:
      containers:
        - name: greeter
          command: [
		"/greeter-srv",
	  ]
          image: yourimage/yourservice
          imagePullPolicy: Always
          ports:
          - containerPort: 8080
            name: greeter-port
          env:
          - name: MICRO_SERVER_ADDRESS
            value: "0.0.0.0:8080"
          - name: MICRO_BROKER
            value: "nats"
          - name: MICRO_BROKER_ADDRESS
            value: "nats-cluster.srv"  # 注意 nats 命名空间 srv
          - name: MICRO_REGISTRY
            value: "etcd"
          - name: MICRO_REGISTRY_ADDRESS
            value: "etcd-cluster-client.srv" # 注意 etcd 命名空间 srv
        - name: health
          command: [
		"/health",
                "--health_address=0.0.0.0:8081",
		"--server_name=greeter",
		"--server_address=0.0.0.0:8080"
	  ]
          image: microhq/health
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 3
            periodSeconds: 3
```
## MicroAPI
要部署Micro API，请使用以下配置。
创建api服务
```
apiVersion: v1
kind: Service
metadata:
  name: micro-api
  namespace: default
  labels:
    name: micro-api
    micro: service
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  selector:
    name: micro-api
    micro: service
  type: LoadBalancer
```
创建部署
```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: micro-api
  labels:
    micro: service
spec:
  replicas: 3
  selector:
    matchLabels:
      name: micro-api
      micro: service
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  template:
    metadata:
      labels:
        name: micro-api
        micro: service
    spec:
      containers:
      - name: api
        env:
        - name: MICRO_SERVER_ADDRESS # api 服务端口 用于健康检测
          value: 0.0.0.0:8989
        - name: MICRO_SERVER_NAME # api 服务名称 用于健康检测
          value: micro
        - name: MICRO_ENABLE_STATS
          value: "true"
        - name: MICRO_BROKER
          value: "nats"
        - name: MICRO_BROKER_ADDRESS
          value: "nats-cluster.srv"  # 注意 nats 命名空间
        - name: MICRO_REGISTRY
          value: "etcd"
        - name: MICRO_REGISTRY_ADDRESS
          value: "etcd-cluster-client.srv" # 注意 etcd 命名空间 srv
        args:
            - "api"
            - "--enable_rpc=true"       # 启用 rpc 模式不使用可以删除
            - "--address=0.0.0.0:8080"
        image: micro/micro
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
      - name: health
          command: [
		"/health",
                "--health_address=0.0.0.0:8081",
		"--server_name=micro",
		"--server_address=0.0.0.0:8989"
	  ]
          image: microhq/health
          livenessProbe:
            httpGet:
              path: /health
              port: 8081
            initialDelaySeconds: 3
            periodSeconds: 3
```
