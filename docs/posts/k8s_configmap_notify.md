---
date: 2023-11-06
readtime: 10
categories:
  - K8s
---



# Kubernetes ConfigMap 实时通知 Pod

在 K8s 中，官方说明 ConfigMap 整体作为卷被 Pod 挂载时，会[自动更新](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-pod-configmap/#mounted-configMaps-are-updated-automatically)。从 ConfigMap 更新到新键映射到 Pod 的**总延迟可能与 kubelet 同步周期（默认为1分钟）+ kubelet 中 ConfigMap 缓存的 TTL（默认为1分钟）**一样长。

官方说明可以**通过更新 Pod 的一个注解来触发立即刷新**。



<!-- more -->

## K8s 更新 ConfigMap 的过程

创建 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test
data:
  mount.info: |
    This is a mount info.
  aa: |
    This is test.
```

并将其挂载到 Pod 中

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /etc/cm
          name: test-cm
  volumes:
    - name: test-cm
      configMap:
        name: test
```

前后两次进入Pod中查看挂载信息：

- `aa`等文件是软链接，不会变；
- configmap 更新时会自动创建新的目录，最后再重命名。

```shell
# 通过 ls -all -i 命令查看
total 4
  2273784 drwxrwxrwx 3 root root   87 Oct 31 10:25 .
132645053 drwxr-xr-x 1 root root 4096 Oct 31 10:25 ..
100673916 drwxr-xr-x 2 root root   34 Oct 31 10:25 ..2023_10_31_10_25_13.340959843
  2335458 lrwxrwxrwx 1 root root   31 Oct 31 10:25 ..data -> ..2023_10_31_10_25_13.340959843
  2273789 lrwxrwxrwx 1 root root    9 Oct 31 10:25 aa -> ..data/aa
  2317395 lrwxrwxrwx 1 root root   17 Oct 31 10:25 mount.info -> ..data/mount.info

# aa 是软链，inode 号不变
root@test:/etc/cm# ll 
total 4
  2273784 drwxrwxrwx 3 root root   87 Oct 31 10:26 .
132645053 drwxr-xr-x 1 root root 4096 Oct 31 10:25 ..
 69055409 drwxr-xr-x 2 root root   34 Oct 31 10:26 ..2023_10_31_10_26_40.305784968
  2339165 lrwxrwxrwx 1 root root   31 Oct 31 10:26 ..data -> ..2023_10_31_10_26_40.305784968
  2273789 lrwxrwxrwx 1 root root    9 Oct 31 10:25 aa -> ..data/aa
  2317395 lrwxrwxrwx 1 root root   17 Oct 31 10:25 mount.info -> ..data/mount.info
```

如果通过 [fsnotity](https://github.com/fsnotify/fsnotify) 监控整个 `/etc/cm`目录的变化，发生的时间如下：
```go
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "CREATE", "file": "/etc/cm/..2023_10_31_10_23_26.701606162"}
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "CHMOD", "file": "/etc/cm/..2023_10_31_10_23_26.701606162"}
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "CREATE", "file": "/etc/cm/..data_tmp"}
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "RENAME", "file": "/etc/cm/..data_tmp"}
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "CREATE", "file": "/etc/cm/..data"}
2023-10-31T10:23:26.098Z	INFO	filewatch/filewatch.go:49	file changed	{"op": "REMOVE", "file": "/etc/cm/..2023_10_31_10_19_52.153989162"}
```



## K8s 更新 ConfigMap 的延迟

kubelet 是用于处理 Master 节点下发到本节点的任务，管理 Pod及Pod 中的容器。

- 宿主机进程：注册到`/etc/systemd/system/kubelet.service`
- `syncFrequency`：默认1min，表示在运行中的**容器与其配置之间执行同步操作的最长时间间隔**；

kubelet 中通过配置`configMapAndSecretChangeDetectionStrategy`指定内部管理器（ConfigMap、Secret） 用来发现对象变化的模式

- **Watch**：默认，使用 watch 操作来观察所关心的对象的变更，**最大延迟为 watch propagation delay**；
  - `whenever a pod is created or updated, we start individual watches for all referenced objects that aren't referenced from other registered pods`

- **Cache**：使用 TTL 缓存来管理来自 API 服务器的对象，**最大延迟为 ttl of cache**；
  - **但当 pod 创建或更新时，其所有的缓存的configmaps都会失效**，即保证获取到最新的 configmap；
  - 通过`node.alpha.kubernetes.io/ttl`来控制
- **Get**：从 API 服务器直接取回必要的对象，**延迟为 0**；



当触发 Pod 修改时，Api Server 给 Kubelet 发通知，**kubelet 会 reconcile pod**，然后获取最新的 ConfigMap 去挂载；



## 通过注解Pod实时更新ConfigMap

通过打开三个 Shell窗口：

- Pod 中执行`ls -all`，通过显示的时间查看 ConfigMap 是否更新；
- `kubectl edit configmap`，修改 ConfigMap 的内容；
- `kubectl edit pod`，修改 Pod 的注解；

**保存完 ConfigMap 的修改之后，立即保存对 Pod 的注解，可以查看到 ConfigMap 会立即更新。**