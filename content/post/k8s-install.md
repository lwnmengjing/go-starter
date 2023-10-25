---
title: "k8s 1.20.0 在centos7 使用 kubeadm 安装"
date: 2021-03-31T10:13:13+08:00
Description: ""
Tags: [k8s,kubeadm]
Categories: [k8s]
DisableComments: false
---
### 系统安装
1. 下载阿里centos7 mini镜像（自带yum阿里镜像源）
2. 制作vmware安装模板
3. 集群规划，利用模板安装节点

<!--more-->
### 安装前准备
#### 禁用swap做准备(每个节点)
```
//注释swap那一行
vi /etc/fstab
#
# /etc/fstab
# Created by anaconda on Wed Nov  4 21:00:49 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=df8ce01c-e0dd-44c9-ae3f-bb051a66a1d6 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

```
#### 规划集群（在192.168.16.131执行）
```
vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.16.131    k8s-master
192.168.16.136    k8s-node1
192.168.16.137    k8s-node2

scp /etc/hosts 192.168.16.136:/etc/hosts
scp /etc/hosts 192.168.16.137:/etc/hosts
```

#### 依赖安装前置配置（每个节点）
注：镜像加速使用自己的地址
```
vi  run.sh
chmod +x run.sh
scp run.sh 192.168.16.136:~/
scp run.sh 192.168.16.137:~/

//copy到自己的run.sh，注意替换镜像加速地址
#!/bin/bash
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
echo vm.swappiness=0 >> /etc/sysctl.conf
swapoff -a
yum install -y ntpdate
ntpdate 0.rhel.pool.ntp.org
yum install -y ntpdath
yum -y install yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["docker镜像加速"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum makecache -y

yum -y install kubelet kubeadm kubectl
systemctl enable kubelet
reboot
```
#### 使用kubeadm安装kubernetes
1. 初始化master
> --kubernetes-version    #指定Kubernetes版本
--image-repository   #由于kubeadm默认是从官网k8s.grc.io下载所需镜像，国内无法访问，所以这里通过--image-repository指定为阿里云镜像仓库地址
--pod-network-cidr    #指定pod网络段
--service-cidr    #指定service网络段
--ignore-preflight-errors=Swap    #忽略swap报错信息

```
kubeadm init --kubernetes-version=v1.20.0 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap
```
2. 记录自己的kubeadm join命令
```
kubeadm join 192.168.16.131:6443 --token xxxx     --discovery-token-ca-cert-hash sha256:xxx
```
3. 等待第一步执行成功
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

scp -r ~/.kube 192.168.16.136:~/
scp -r ~/.kube 192.168.16.137:~/
```
4. 安装网络组件(192.168.16.131)
```
方法1(不需要梯子)：
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
方法2(自备梯子下载):
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f  kube-flannel.yml
```

5. 加入node节点（分别在192.168.16.136(137)执行）
```
kubeadm join 192.168.16.131:6443 --token xxxx     --discovery-token-ca-cert-hash sha256:xxx
```
6. 安装dashboard(可自行改为NodePort暴露)
```
kubectl create -f  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/recommended.yam
//授权
vi admin-user-sa-rbac.yaml
kubectl apply -f admin-user-sa-rbac.yaml
//获取token
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

admin-user-sa-rbac.yaml 内容

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

```
