---
date: 2023-12-25
draft: true
readtime: 10
categories:
  - K8s
---

# K8s Configmap 挂载避免符号链接

在 k8s 中挂载 configmap 时，默认情况下，会以符号链接的形式存在。

在某些场景下，如 Pod 挂载 `.ssh` 进行免密时，由于`.ssh`的特殊权限，因此不能以符号链接的形式存在，否则不能 ssh 免密。

此时，可以使用 `subpath` 进行挂载。

<!-- more -->