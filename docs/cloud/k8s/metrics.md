# 资源监控



## k8s 内置的资源监控指标

指标是由轻量级的、短期、内存存储的 [metrics-server](https://github.com/kubernetes-sigs/metrics-server) 收集， 并通过 `metrics.k8s.io` 公开

- metrics-server 发现集群中的所有节点，并且查询每个节点的 kubelet 以获取 CPU 和内存使用情况。

## 参考文档

1. [资源监控工具 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-usage-monitoring/)

