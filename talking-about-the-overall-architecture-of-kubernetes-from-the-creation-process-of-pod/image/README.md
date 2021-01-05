# 从Pod的创建过程谈谈Kubernetes的整体架构

## 序言
Pod是Kubernetes集群中能够被创建和管理的最小部署单元，本文通过讲解Pod的创建过程和实现原理，来探探Kubernetes的整体架构。我们先通过Yaml文件来展示Pod的定义：
```yaml
# pod.yaml
apiVersion: v1          # KubernetesAPI版本
kind: Pod               # Kubernetes资源类型
metadata:               # 元数据描述
  name: pod_name        # 指定Pod的名称
  namespace: default    # 指定Pod运行的命名空间
  labels:               # 指定pod标签
    app: pod_label
  annotations:          # 指定Pod的注释
    version: v0.0.1
spec:                   # 规范，用于描述对象的详细配置
  containers:
  - name: c1            # 容器的名称
    image: nginx        # 创建容器所使用的镜像
    ports:
    - containerPort: 80 # 应用监听的端口
  - name: c2            # 容器的名称
    image: busybox      # 创建容器所使用的镜像
    command:            # 容器启动命令
      - "bin/bash"
      - "-c"
      - "sleep 100"
```
执行以下命令：
```
kubectl create -f pod.yaml
```
稍等片刻我们可以通过执行命令`kebectl get pods`便可看到以下结果：
```
NAME                      READY     STATUS    RESTARTS   AGE
pod_name                  2/2       Running   0          1s
```
好奇心重的同学可能会问，`kubectl create`从执行到`Pod`会创建完成，这中间件到底发生了什么呢？整个生命周期在流转的过程中经过了哪些组件？让我们从`kubectl`说起...

## kubectl

## kube-api-server

## kube-controller-manager

## kube-scheduler

## etcd

## kubelet

## kube-proxy 

## 架构