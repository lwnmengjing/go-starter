---
title: "go-admin部署到istio平台"
date: 2021-03-31T10:25:43+08:00
Description: ""
Tags: [go-admin,istio]
Categories: [go-admin,istio]
DisableComments: false
---
### 部署istio环境参考[istio](https://istio.io/latest/zh/docs/setup/install/)官网
### 创建独立命名空间go-admin，自动注入 sidecar
```shel
kubectl create namespace go-admin
kubectl label namespace go-admin istio-injection=enabled
```
<!--more-->
### 创建配置configmap
```shell
kubectl create configmap settings-admin --from-file=config/settings.yml -n go-admin
```
### pv及pvc根据自己到需求调整
```shell
kubectl apply -f storage.yml -n go-admin
#storage.yml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: go-admin
  namespace: go-admin
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "1Mi"
  volumeName:
  storageClassName: nfs-csi
```
### 部署后端服务v1版本[dev分支](https://github.com/go-admin-team/go-admin/tree/dev)
#### 配置
```shell
kubectl apply -f deploy.yml -n go-admin
# deploy.yml
---
apiVersion: v1
kind: Service
metadata:
  name: go-admin
  namespace: go-admin
  labels:
    app: go-admin
    service: go-admin
spec:
  ports:
  - port: 8000
    name: http
    protocol: TCP
  selector:
    app: go-admin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-admin-v1
  namespace: go-admin
  labels:
    app: go-admin
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-admin
      version: v1
  template:
    metadata:
      labels:
        app: go-admin
        version: v1
    spec:
      containers:
      - name: go-admin
        image: registry.cn-shanghai.aliyuncs.com/go-admin-team/go-admin:v1.2.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: go-admin
          mountPath: /temp
        - name: go-admin
          mountPath: /static
        - name: go-admin-config
          mountPath: /config/
          readOnly: true
      volumes:
      - name: go-admin
        persistentVolumeClaim:
          claimName: go-admin
      - name: go-admin-config
        configMap:
          name: settings-admin
---
```
### 创建前端nginx配置configmap
```shell
kubectl create configmap nginx-frontend --from-file=default.conf -n go-admin
#default.conf
server {
  listen       80;
  listen  [::]:80;
  server_name  localhost;
  
  #charset koi8-r;
  #access_log  /var/log/nginx/host.access.log  main;

  location / {
      root   /usr/share/nginx/html;
      index  index.html index.htm;
  }
}
```
### 部署前端服务v1版本[dev分支](https://github.com/go-admin-team/go-admin-ui/tree/dev)
```shell
kubectl apply -f deploy.yml
# deploy.yml
---
apiVersion: v1
kind: Service
metadata:
  name: go-admin-ui
  namespace: go-admin
  labels:
    app: go-admin-ui
    service: go-admim-ui
spec:
  ports:
    - port: 80
      name: http
      protocol: TCP
  selector:
    app: go-admin-ui
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-admin-ui-v1
  namespace: go-admin
  labels:
    app: go-admin-ui
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-admin-ui
      version: v1
  template:
    metadata:
      labels:
        app: go-admin-ui
        version: v1
    spec:
      containers:
        - name: go-admin-ui
          image: registry.cn-shanghai.aliyuncs.com/go-admin-team/go-admin-ui:v1.2.2
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: frontendconf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
              readOnly: true
      volumes:
        - name: frontendconf
          configMap:
            name: nginx-frontend
---
```

### 创建dr
```shell
kubectl apply -f destination-go-admin.yaml -n go-admin
#destination-go-admin.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: go-admin-ui
  namespace: go-admin
spec:
  host: go-admin-ui
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: go-admin
  namespace: go-admin
spec:
  host: go-admin
  subsets:
  - name: v1
    labels:
      version: v1

```
### 创建gateway和vs
> 域名改成实际域名
```shell
kubectl apply -f go-admin-gateway.yml -n go-admin
#go-admin-gateway.yml
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: go-admin-gateway
  namespace: go-admin
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "go-admin.xxxxxx.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: go-admin-ui
  namespace: go-admin
spec:
  hosts:
  - "*"
  gateways:
  - go-admin-gateway
  http:
  - match:
    - uri:
        prefix: /api
    - uri:
        prefix: /login
    route:
    - destination:
        host: go-admin
        subset: v1
        port:
          number: 8000
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: go-admin-ui
        subset: v1
        port:
          number: 80
```

### 创建访问外部数据库ServiceEntry
> 根据自己的需求，将hosts改为自己的数据链接地址
```shell
kubectl apply -f serviceEntry.yaml
#serviceEntry.yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: db-mysql
  namespace: go-admin
spec:
  hosts:
  - "192.168.16.69"
  ports:
  - number: 3306
    name: tcp
    protocol: tcp
  location: MESH_EXTERNAL
  resolution: DNS
```
### kiali展示
![在这里插入图片描述](https://img-blog.csdnimg.cn/202102011404076.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70#pic_center)
