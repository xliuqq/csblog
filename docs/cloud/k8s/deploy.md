# 安装部署



## 高可用架构

API Server来说，其高可用性是通过多实例部署、负载均衡、etcd集群、健康检查和监控等多种机制共同实现的，而不是通过leader election来决定哪个实例可以对外提供服务。

leader election机制更适用于需要确保同一时刻只有一个实例执行特定任务的场景，如kube-scheduler和kube-controller-manager。

### 堆叠（Stacked）etcd 拓扑 

![堆叠的 etcd 拓扑](.pics/deploy/kubeadm-ha-topology-stacked-etcd.svg)

### 外部 etcd 拓扑

![外部 etcd 拓扑](.pics/deploy/kubeadm-ha-topology-external-etcd.svg)

