---
title: "k8s 1.20.0安装周边组件及兼容性"
date: 2021-03-31T10:15:41+08:00
Description: ""
Tags: [k8s]
Categories: [k8s]
DisableComments: false
---
### 安装metrics-server
> 由于kubeadm默认不安装metrics-server
<!--more-->
```shell
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
#在metrics-server-deployment.yaml 中添加如下command部分
#...code
		image: k8s.gcr.io/metrics-server/metrics-server:v0.4.1
        imagePullPolicy: IfNotPresent
        command:
          - /metrics-server
          - --kubelet-preferred-address-types=InternalIP
          - --kubelet-insecure-tls
 
kubectl apply -f components.yaml
```
```shell
[root@k8s-master metrics]# kubectl top nodes
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master   261m         6%     976Mi           12%
k8s-node1    76m          1%     445Mi           5%
k8s-node2    70m          1%     435Mi           5%

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201211142232379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)
### 创建持久存储卷
#### 安装nfs(192.168.16.69)
1. 如果没有关闭防火墙，请打开端口111、2049，或者关闭防火墙
```shell
#firewalld
firewall-cmd  --permanent    --add-port=111/tcp
firewall-cmd  --permanent    --add-port=111/udp
firewall-cmd  --permanent    --add-port=2049/tcp
firewall-cmd  --permanent    --add-port=2049/udp
#iptables
vi /etc/sysconfig/iptables
#添加一下内容
-A INPUT -p tcp -m state --state NEW -m tcp --dport 111 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 111 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2049 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 2049 -j ACCEPT
service iptables restart
```
2. 安装 NFS 服务器所需的软件包
```shell
yum install -y nfs-utils
```
3. 编辑exports文件，允许指定网段客户端挂载
```shell
mkdir -p /data/nfs
vi /etc/exports
#添加
/data/nfs/ 192.168.16.0/24(rw,all_squash,anonuid=0,anongid=0)
```
3. 启动nfs服务
> 先为rpcbind和nfs做开机启动：(必须先启动rpcbind服务)，然后分别启动rpcbind和nfs服务：
```shell
systemctl enable rpcbind.service
systemctl enable nfs-server.service

systemctl start rpcbind.service
systemctl start nfs-server.service

#使配置生效
exportfs -r
```
4. 验证
```shell
exportfs
#可以查看到已经ok
/data/nfs/ 192.168.16.0/24
```
5. 自行验证挂载

### 安装helm v3
1. 下载并安装helm：
```shell
curl -SLO https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
tar -zxvf helm-v3.4.2-linux-amd64.tar.gz
chmod +x linux-amd64/bin/helm && cp linux-amd64/bin/helm /usr/local/bin/
```
2. 由于helm没有自带仓库镜像，拉取镜像
```shell
#墙内网络使用国内镜像
helm repo add stable https://apphub.aliyuncs.com/stable
```
### 安装ingress-controller
> `ingress-controller`选择很多，由于搭建在私有环境，我选择`ingress-nginx`，也是k8s官方支持的controller，并且目前不是生产阶段，先不考虑lb部署方式，直接使用节点IP。上一步已经安装`helm`，那就简单粗暴，一把梭，使用`helm`直接部署
1.  使用helm部署ingress-controller
```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```
2. 设置ingress-nginx-controller的externalIPs，更新...
```yaml
  clusterIP: 10.100.9.184
  clusterIPs:
    - 10.100.9.184
  type: LoadBalancer
  externalIPs:
    - 192.168.16.136
    - 192.168.16.137
    - 192.168.16.131
  sessionAffinity: None
  externalTrafficPolicy: Cluster
```
3. 修改controller deploy的image，我是把镜像先拉取到了harbor中
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201215112630644.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)


4. 验证成功
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201215110727755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)
### dashboard改为ingress对外暴露
1. 创建ingress yaml文件`ingress-nginx-kubernetes-dashboard.yaml`
> 这个地方的`networking.k8s.io/v1beta1`为之前版本，会有告警，暂时先忽略，等后期理解加深了在回来修复这里
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-nginx-kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 开启use-regex，启用path的正则匹配
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    #nginx.ingress.kubernetes.io/secure-backends: "true" //好像是版本0.20.0发布后被删除，请使用下面这行
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - "k8s.local"
    - "*.k8s.local"
    secretName: kubernetes-dashboard-certs
  rules:
  - host: dashboard.k8s.local
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443

```
2. 执行部署
```shell
kubectl apply -f ingress-nginx-kubernetes-dashboard.yaml
```
3. 修改系统hosts文件，增加一行
```bash
192.168.16.131 dashboard.k8s.local
```
4. 使用域名访问验证
> 本来就是私有环境，自签证书默认使用k8s创建的证书，如有需要请提前修改

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201215111541270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)

### 兼容性问题
> 1. 动态创建nfs类存储卷 不兼容，可以手动创建pv挂载
---
### 兼容问题解决
> 从k8s 1.14开始，k8s实现了csi规范，之前版本都能使用原先的nfs client正常动态创建pv，但是k8s1.20版本开始，pod中创建pv报错，应该是只保留了csi规范。所以，我们需要在k8s集群安装nfs driver支持。
#### 安装`csi-driver-nfs`
1. 使用helm安装`csi-driver-nfs`
```shell
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install --name csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system
#如果网络有问题，可以直接下载https://github.com/kubernetes-csi/csi-driver-nfs，cd charts/latest && helm install csi-driver-nfs ./csi-driver-nfs -n kube-system
```
2. 创建storageClass
```shell
vim storageclass-nfs.yaml
#storageclass-nfs.yaml内容
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi-69
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.16.69
  share: /data/nfs/pv
reclaimPolicy: Retain  # only retain is supported
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
kubectl apply -f storageclass-nfs.yaml
```
3. 创建pvc验证动态创建pv
```shell
vim pvc-nfs-csi-dynamic.yaml
#pvc-nfs-csi-dynamic.yaml内容
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  storageClassName: nfs-csi-69
kubectl apply -f pvc-nfs-csi-dynamic.yaml
```
```shell
#验证
[root@k8s-master nfs-server]# kubectl apply -f pvc-nfs-csi-dynamic.yaml
persistentvolumeclaim/pvc-nfs-dynamic created
[root@k8s-master nfs-serjjver]# kubectl get pv|grep nfs-csi
pvc-3ba895b7-4083-40fe-afa6-5c4c90299f49   1Mi        RWX            Retain           Bound    default/pvc-nfs-dynamic   nfs-csi              18s
```
