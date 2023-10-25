---
title: "Istio代理（Envoy）proxy-wasm扩展(oci)"
date: 2021-03-31T10:30:01+08:00
Description: ""
Tags: [istio]
Categories: [istio]
DisableComments: false
---
### 场景描述
部署在istio的http服务需要拦截请求做处理，使用proxy-wasm作为envoy的扩展，harbor2作为仓库
<!--more-->

### 安装WebAssembly Hub CLI
安装cli
```shell
curl -sL https://run.solo.io/wasme/install | sh
export PATH=$HOME/.wasme/bin:$PATH
```
验证cli
```shell
wasme --version

wasme version 0.0.32
```
### 初始化filter项目
> 可以修改assemblyscript的代码实现需要的功能
```shell
wasme init ./testfilter
Use the arrow keys to navigate: ↓ ↑ → ← 
? What language do you wish to use for the filter: 
    cpp
    rust
  ▸ assemblyscript
    tinygo

✔ assemblyscript
Use the arrow keys to navigate: ↓ ↑ → ← 
? With which platforms do you wish to use the filter?: 
  ▸ istio:1.5.x, istio:1.6.x, istio:1.7.x, gloo:1.6.x, istio:1.8.x
```
### build项目推送wasm oci到harbor
```shell
wasme build assemblyscript -t 192.168.16.90/wasm/testfilter:v0.1 .
wasme push 192.168.16.90/wasm/testfilter:v0.1 --insecure --username=admin --password=Harbor12345
```
> 查看harbor仓库wasm项目下hello库
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206114406947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)

### 部署Wasme CRDs
```shell
# step1 deploy crds
kubectl apply -f https://github.com/solo-io/wasme/releases/latest/download/wasme.io_v1_crds.yaml
# step2 deploy Operator components
kubectl apply -f https://github.com/solo-io/wasme/releases/latest/download/wasme-default.yaml
```

### 部署proxy-wasm扩展
```shell
# vim wasm.yaml
apiVersion: wasme.io/v1
kind: FilterDeployment
metadata:
  labels:
    app: wasme-test-app
    app.kubernetes.io/name: wasme-test-app
  name: myfilter
  namespace: go-admin
spec:
  deployment:
    istio:
      kind: Deployment
  filter:
    config:
      '@type': type.googleapis.com/google.protobuf.StringValue
      value: world
    image: 192.168.16.90/wasm/testfilter:v0.1

```

### 验证
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206112255647.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206112040353.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDgxNDg2Nw==,size_16,color_FFFFFF,t_70)
