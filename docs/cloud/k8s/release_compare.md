# K8s的可选发行版

## k8s/MiniKube（开发环境）

> minikube quickly sets up a local Kubernetes cluster on macOS, Linux, and Windows. We proudly focus on helping **application developers and new Kubernetes users**.

- 基于 VM/Container 启动的 K8s 集群；
- 支持多节点集群（在一个物理节点启动两个VM组成K8s），而不是多个物理节点组成K8s集群；

### 安装

> https://minikube.sigs.k8s.io/docs/start/
>
> https://github.com/kubernetes/minikube/releases/latest

#### Linux

```shell
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

#### Windows

**安装 Hyper-V**

> 【控制面板】->【程序】->【打开和关闭 windows 功能】-> 勾选 Hyper-V 的选项（包括所有子项）-> 重启电脑

**启用 Hyper-V**

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

```shell
minikube start --driver=hyperv 
# To make hyperv the default driver:
minikube config set driver hyperv
```

### 使用

```bash
# minikube 会自动下载 kubectl
alias kubectl="minikube kubectl --"
# 启动 minikube，使用国内的镜像
minikube start --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --base-image=registry.cn-hangzhou.aliyuncs.com/google_containers/kicbase:v0.0.44
```

### 本地镜像使用

> [minikube推送镜像到本机](https://minikube.sigs.k8s.io/docs/handbook/pushing/)

使用 docker driver，因为 docker daemon 存在，可以共享 docker sock；

podman driver 需要配合 cri-o 而非 docker ，因为 podman 不存在 daemon 进程。



## Kind

> Kubernetes IN Docker - **local clusters for testing Kubernetes**.
>
> kind is a tool for running local Kubernetes clusters using Docker container "nodes". kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

测试环境



## [MicroK8s](https://github.com/canonical/microk8s)

> **MicroK8s** is a low-ops, minimal production Kubernetes.
>
> MicroK8s is a small, fast, single-package Kubernetes for developers, **IoT and edge**.



## [K3s](https://github.com/k3s-io/k3s/)

> Lightweight Kubernetes. The certified Kubernetes distribution built for **IoT & Edge computing**.

