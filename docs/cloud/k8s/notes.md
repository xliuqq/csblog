# 注意事项



## 本地磁盘限制

> 防止将宿主机的磁盘撑满，导致宿主机不可用。

### ephemeral-storage

ephemeral-storage 包括：

- `emptyDir` volumes, except *tmpfs* `emptyDir` volumes
- directories holding node-level logs
- writeable container layers

> 在每个Kubernetes的节点上，kubelet的根目录(默认是/var/lib/kubelet)和日志目录(/var/log)保存在节点的主分区上，这个分区同时也会被**Pod的EmptyDir类型的volume、容器日志、镜像的层、容器的可写层所占用**。
>
> - kubelet会统计当前节点的主分区的可分配的磁盘资源，或者可以覆盖节点上kubelet的配置来自定义可分配的资源。
> - 在创建Pod时会根据存储需求调度到满足存储的节点，在Pod使用超过限制的存储时会对其做**驱逐**的处理来保证不会耗尽节点上的磁盘空间。

注意事项：

-  如果k8s的根目录（kubelet root-dir）跟容器镜像存储根目录（/var/lib/{docker}）是单独挂载的磁盘（非root盘），则通过 kubectl 查看 node 的 ephemeral-storage 只会显示根分区(/) 的存储空间。
   - The kubelet will only track the root filesystem for ephemeral storage. OS layouts that mount a separate disk to `/var/lib/kubelet` or `/var/lib/containers` will not report ephemeral storage correctly.
   - 在申请资源的时候，可能会出问题（因为总的资源量结果不对）

-  K8s 1.22 版本及之后，允许[**容器镜像存储根目录（/var/lib/{docker}）单独挂载盘**](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#configurations-for-local-ephemeral-storage)，如果镜像可写层超过限制，可以正常被驱除；

使用示例 Pod：

```yaml
resources:
  requests:
    cpu: 1
    memory: 2048Mi
    # k8s 1.8开始引入的特性，限制容器存储空间的使用
    ephemeral-storage: 2Gi
  limits:
    cpu: 2
    memory: 2048Mi
    ephemeral-storage: 5Gi
```

**当ephemeral-storage超出限制时，kublet会驱除当前容器**

> **Pod 内的容器不会被重启**（不会受 restartPolicy 影响，也不会受存活等探针影响），因此需要配合 deployment 等使用，由 deployment 重新创建 Pod；
>
> - 已经 restart 过，但是有错误信息，`reason`状态导致不会继续重启（k8s 1.22版本），参考[Pod驱除](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/)；

- `get pods -o yaml`信息如下：

```yaml
status:
  - lastProbeTime: null
    lastTransitionTime: "2024-02-04T09:31:31Z"
    reason: PodFailed
    status: "False"
    type: Ready
  - ...
  containerStatuses:
  - image: nginx
    ...
    ready: false
    restartCount: 1
    started: false
    state:
      terminated:
        exitCode: 137
        finishedAt: null
        message: The container could not be located when the pod was terminated
        reason: ContainerStatusUnknown
        startedAt: null
  message: 'Pod ephemeral local storage usage exceeds the total limit of containers 1Gi. '
  phase: Failed
  qosClass: Burstable
  reason: Evicted
```

查看事件时如下：

```shell
  Warning  Evicted    27s    kubelet            Pod ephemeral local storage usage exceeds the total limit of containers 1Gi.
  Normal   Killing    27s    kubelet            Stopping container nginx
```





##  Event 持久化

> Event作为kubernetes的一个对象资源，记录了集群运行所遇到的各种大事件,有助于排错，但大量的事件如果都存储在etcd中，会带来较大的性能与容量压力，所以 **etcd 中默认只保存最近1小时**的。

k8s集群规模的增加，集群内的object数量也与日俱增，那么events的数量也会伴随其大量增加，那么当用户请求这些events的时候apiserver的负载压力就会增加，很可能造成 apiserver 处理请求延迟，可以将 events 持久化，通过 webhook 拦截用户请求。

### [eventrouter](https://github.com/openshift/eventrouter)

> A **simple** introspective kubernetes service that forwards events to a specified sink.

将 `ADD`或者 `Update`的事件，以及对应的 `newEvent`和 `oldEvent`，发送给Sink。

- 通过 `informers.NewSharedInformerFactory`进行监听和处理；
- 统计 `Normal`，`WARNING`相同的次数，给prometheus使用；

### [kubewatch](https://github.com/robusta-dev/kubewatch)

> **kubewatch** is a Kubernetes watcher that currently publishes notification to available collaboration hubs/notification channels. Run it in your k8s cluster, and you will get event notifications through webhooks.

supported webhooks:

- slack, slackwebhook, hipchat, mattermost, flock, cloudevent
- **webhook**
- smtp

可以配置监听不同的资源，

```yaml
resource:
  deployment: false
  replicationcontroller: false
  replicaset: false
  daemonset: false
  services: true
  pod: true
  job: false
  node: false
  clusterrole: false
  clusterrolebinding: false
  serviceaccount: false
  persistentvolume: false
  namespace: false
  secret: false
  configmap: false
  ingress: false
```

[kube-eventer](https://github.com/AliyunContainerService/kube-eventer)

> kube-eventer emit kubernetes events to sinks.

支持的Sink

| Sink Name                                                    | Description                       |
| ------------------------------------------------------------ | --------------------------------- |
| [dingtalk](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/dingtalk-sink.md) | sink to dingtalk bot              |
| [sls](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/sls-sink.md) | sink to alibaba cloud sls service |
| [elasticsearch](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/elasticsearch-sink.md) | sink to elasticsearch             |
| [honeycomb](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/honeycomb-sink.md) | sink to honeycomb                 |
| [influxdb](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/influxdb-sink.md) | sink to influxdb                  |
| [kafka](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/kafka-sink.md) | sink to kafka                     |
| [mysql](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/mysql-sink.md) | sink to mysql database            |
| [wechat](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/wechat-sink.md) | sink to wechat                    |
| [webhook](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/webhook-sink.md) | sink to webhook                   |
| [mongodb](https://github.com/AliyunContainerService/kube-eventer/blob/master/docs/en/mongodb-sink.md) | sink to mongodb                   |
