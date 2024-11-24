# Containerd

> [An open and reliable container runtime. ](https://github.com/containerd/containerd)CNCF gradulated project.

containerd 是一个工业级标准的容器运行时，它强调**简单性**、**健壮性**和**可移植性**，其功能包括：

- 管理容器的生命周期（从创建容器到销毁容器）
- 拉取/推送容器镜像
- 存储管理（管理镜像及容器数据的存储）
- 调用 runc 运行容器（与 runc 等容器运行时交互）
- 管理容器网络接口及网络

## 架构

![img](.pics/containerd/containerd_arch.webp)

## 配置

containerd 可以修改 http://docker.io 对应的 endpoint（默认为https://registry-1.docker.io），而 docker 无法修改。