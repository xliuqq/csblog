# [OpenFaas](https://docs.openfaas.com/)

> Serverless Functions, Made Simple. OpenFaaS® makes it simple to deploy both functions and existing code to Kubernetes.

**OpenFaaS 函数在容器中运行**，并且每个容器必须遵守**简单的约定** ：它作为监听在**预设端口（默认为 8080）上的 HTTP 服务器**，**临时存储**并且是**无状态**的。

- 支持 K8s / OpenShift / Faasd（单机）；

## 原理

> 通过添加Watchdog程序(一个小型的Golang HTTP服务器)，将任何Docker镜像都添加到一个serverless函数中。

### [经典 watchdog](https://github.com/openfaas/classic-watchdog)

- 每次函数调用都启动单独的进程；
- **任何使用 \*stdio\* 流进行 I/O 处理的程序（包括最喜欢的 CLI 工具）都可以部署为 OpenFaaS 函数**

![img](.pics/openfaas/simple_watch_dog.png)

### [反向代理 watchdog](https://github.com/openfaas/of-watchdog)

- watchdog 只创建一次函数的进程，并将其当成（长期运行的）上游服务器；

![img](.pics/openfaas/proxy_watch_dog.png)

### 其他运行时模式

问题1：**经典**运行时模式在将函数结果发送回调用方之前缓冲了函数的整个响应。但如果响应的大小超出了容器的内存怎么办？

- OpenFaaS 提供了[Streaming模式，该模式仍然为每个调用创建进程，但添加了**响应流**](https://github.com/openfaas/of-watchdog/blob/a0289419078824f0a070860f84a6b383eb4f2169/README.md#3-streaming-fork-modestreaming---default)。

问题2：经典运行模式如何提供文件服务；

- OpenFaaS 提供了[Static模式，提供HTTP文件服务](https://github.com/openfaas/of-watchdog#4-static-modestatic)



## 架构

![img](.pics/openfaas/openfaas_arch.png)

### Kubernetes 上的 OpenFaaS（faas-nets）

- API 网关成为标准（部署+服务）对；
- 每个函数也成为（部署+服务）对；
- Kubernetes 作为一个数据库工作，比如列出所有的部署函数列表；

![img](.pics/openfaas/openfaas_k8s.png)

### Containerd 上的 OpenFaaS（faasd，单机）

- 被设计为在 **IoT 设备上或 VM 中运行**；
- 使用 containerd 的原生 `pause` （通过*cgroup freezers*）和超快速函数冷启动快速扩展到零；
- containerd 和 faasd 作为 systemd 服务进行管理，因此会自动处理日志、重启等；
- 没有 Kubernetes DNS，但 faasd 确保 DNS 在函数之间共享以简化互操作；
- containerd 扮演着数据库的角色（如果服务器挂了，状态丢失，函数需要重新部署）；
- 没有开箱即用的高可用性或水平扩展（参见 [issue/225](https://github.com/openfaas/faasd/issues/225)）

![openfaas_fassd.png](.pics/openfaas/openfaas_fassd.png)