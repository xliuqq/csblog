---
date: 2023-12-25
readtime: 10
categories:
  - K8s
---

# K8s Configmap 挂载避免符号链接

在 k8s 中挂载 configmap 时，默认情况下，会以符号链接的形式存在。

在某些场景下，如 Pod 挂载 `.ssh` 进行免密时，由于`.ssh`的特殊权限，因此不能以符号链接的形式存在，否则不能 ssh 免密。

此时，可以使用 `subpath` 进行挂载。

<!-- more -->



## 如何配置 subpath 避免符号链接

> subpath 挂载 configmap 时，当 configmap 变更时，pod 内的文件无法得到更新。

configmap 不使用 subpath 挂载时

```yaml
spec:
  containers:
    volumeMounts:
    - name: ssh-config
      mountPath: /root/.ssh
  volumes:
    - name: ssh-config
    configMap:
      name: ssh-config-cm
      items:
        - key: PRIVATE_KEY
          path: id_rsa
        - key: PUBLIC_KEY
          path: id_rsa.pub
        - key: AUTHORIZED_KEYS
          path: authorized_keys
        - key: CONFIG
          path: config
```

挂载的目录结构如下面所示：

- 挂载的文件 `id_rsa` 都是符号链接；
- `/root/.ssh`如果原先的pod的镜像中存在该路径，则会被覆盖；

```shell
# 通过 ls -all -i /root/.ssh 命令查看
total 4
  2273784 drwxrwxrwx 3 root root   87 Oct 31 10:25 .
132645053 drwxr-xr-x 1 root root 4096 Oct 31 10:25 ..
100673916 drwxr-xr-x 2 root root   34 Oct 31 10:25 ..2023_10_31_10_25_13.340959843
  2335458 lrwxrwxrwx 1 root root   31 Oct 31 10:25 ..data -> ..2023_10_31_10_25_13.340959843
  2273789 lrwxrwxrwx 1 root root    9 Oct 31 10:25 id_rsa -> ..data/id_rsa
  2317395 lrwxrwxrwx 1 root root   17 Oct 31 10:25 id_rsa.pub -> ..data/id_rsa.pub
  ... ...
```

使用 configmap subpath 挂载时：

```yaml
spec:
  containers:
    volumeMounts:
    - name: ssh-config
      mountPath: /root/.ssh
      # use subpath to avoid soft link
      subPath: .ssh
  volumes:
    - name: ssh-config
    configMap:
      name: ssh-config-cm
      defaultMode: 0600
      items:
        - key: PRIVATE_KEY
          path: .ssh/id_rsa
        - key: PUBLIC_KEY
          path: .ssh/id_rsa.pub
        - key: AUTHORIZED_KEYS
          path: .ssh/authorized_keys
        - key: CONFIG
          path: .ssh/config
```

挂载的目录结构如下面所示：

- `id_rsa`等不再是符号链接，而是文件；

```shell
# 通过 ls -all -i /root/.ssh 命令查看
total 4
  2273784 drwxrwxrwx 3 root root   87 Oct 31 10:25 .
132645053 drwxr-xr-x 1 root root 4096 Oct 31 10:25 ..
  2273789 -rw------- 1 root root    9 Oct 31 10:25 id_rsa
  2317395 -rw------- 1 root root   17 Oct 31 10:25 id_rsa.pub
  ... ...
```

