---
date: 2023-12-27
readtime: 10
categories:
  - K8s
---



# K8s Pod 如何配置免密

MPI Operator 在执行 MPI 作业时，通过 [SSH 的方式](https://github.com/kubeflow/mpi-operator/blob/master/proposals/scalable-robust-operator.md#design)，因此需要在不同的 MPI Workers Pod间配置免密。

本文通过源码分析，探究如何在 K8s Pod 间配置免密。



<!-- more -->

## SSH 免密

一般常规的免密操作，都是在不同的节点上，通过 `ssh-keygen` 生成公钥和私钥，然后将所有节点的公钥都写到各个节点的`authorized_keys`文件中。

但是查看 [volcano mpi](https://github.com/volcano-sh/volcano/blob/ea04e652273d4714baedc1d1fffa736b73099d61/pkg/controllers/job/plugins/ssh/ssh.go#L37) 的实现，可以知道**Pod 使用同样的私钥/公钥，可以互相免密访问**。

因此，只需生成同样的 `id_rsa`和 `id_rsa.pub`，然后将 `authorized_keys`的内容设为`id_rsa.pub`一致。

最后配置下`config`文件：

```ini
# 第一次访问时，无需确认，直接连接
StrictHostKeyChecking no
UserKnownHostsFile /dev/null
# pod 间无法通过 hostname 直接访问对方的，因此需要 service
Host pod-hostname-1
  HostName  pod-hostname-1.service-name
Host pod-hostname-2
  HostName  pod-hostname-2.service-name
```

将 `id_rsa,id_rsa.pub,authorized_keys,config`作为 configmap的内容。

## Pod 主机名解析

pod 间无法通过 hostname 直接访问对方的，因此需要 service：

- 可以参考 [Pod 间通信](https://kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/)；

hostname 必须是规则化，才可以静态生成所有的主机名：

- 可以用 Statefulset 或者 IndexJob 来生成有规则的 hostname。



## Pod 挂载 SSH ConfigMap

想要免密，则 Pod 上挂载相应的configmap 即可。

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

