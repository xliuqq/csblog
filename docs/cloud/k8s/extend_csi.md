# CSI

> Container Storage Interface

开源的一般有

-  [Longhorn](https://www.rancher.cn/longhorn/)：云原生分布式块存储；
- Rook：Ceph，支持块、文件、对象；
- GlusterFS、Lustre等。

对象存储不需要PV/PVC来做资源抽象，应用可以直接访问和使用。

- 对象存储的CSI 一般使用S3协议，并挂载成文件系统（如 [s3fs-fuse](https://github.com/s3fs-fuse/s3fs-fuse)），但是兼容性和效率有限制（如`append`, `list`等）。
- JuiceFS 使用对象存储和元数据分离，可以避免类似的问题。

## 服务

CSI 插件主要有3个服务：

- **CSI Node**：对主机上的存储卷进行相应的操作，负责将 Volume mount 到 pod 中，以 DaemonSet 形式部署在每个 node 中；
- **CSI Controller**：从存储服务端角度对存储卷进行管理和操作，**负责 Volume 的管理**，以 StatefulSet 形式部署。
- **CSI Identify**：用来获取插件的信息，检查插件的状态。

## 接口规范

### Indentity接口

用于提供 CSI driver 的身份信息，Controller 和 Node 都需要实现。

```go
service Identity {
  // 返回插件的名字和版本
  rpc GetPluginInfo(GetPluginInfoRequest) returns (GetPluginInfoResponse) {}
  // 返回插件的功能点，是否支持存储卷创建、删除等功能，是否支持存储卷挂载的功能
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest) returns (GetPluginCapabilitiesResponse) {}
  // 返回插件的健康状态(是否在运行中)
  rpc Probe (ProbeRequest) returns (ProbeResponse) {}
}
```

### Controller接口

属于 K8s的 Volume Controller 的逻辑，即属于 Master 节点的一部分。

- **CreateVolume**：创建一个存储卷(如EBS盘)
  -  **无需在宿主机上执行**的操作，k8s 通过调用该方法创建底层存储

- **DeleteVolume**：删除一个已创建的存储卷(如EBS盘)
- **ControllerPublishVolume**：将一个已创建的存储卷，挂载(attach)到指定的节点上
- **ControllerUnpublishVolume**：从指定节点上，卸载(detach)指定的存储卷
- ValidateVolumeCapabilities：返回存储卷的功能点，如是否支持挂载到多个节点上，是否支持多个节点同时读写
- ListVolumes：返回所有存储卷的列表
- GetCapacity：返回存储资源池的可用空间大小
- ControllerGetCapabilities：返回controller插件的功能点，如是否支持GetCapacity接口，是否支持snapshot功能等.

### Node接口

- **NodeStageVolume**：如果存储卷没有格式化，首先要格式化。然后**把存储卷mount到一个临时的目录**(这个目录通常是节点上的一个全局目录)。再通过NodePublishVolume将存储卷mount到pod的目录中。**mount过程分为2步，原因是为了支持多个pod共享同一个volume(如NFS)**。

- **NodePublishVolume**：将存储卷**从临时目录mount到目标目录(pod目录)**，Pod创建时才会进行该操作；

- **NodeUnstageVolume**：NodeStageVolume的逆操作，将一个存储卷从临时目录umount掉（一个节点同一个PVC没有Pod挂载时才会操作）

- **NodeUnpublishVolume**：将存储卷从pod目录umount掉，Pod删除时会执行

- NodeGetId：返回插件运行的节点的ID

- NodeGetCapabilities：返回Node插件的功能点，如是否支持`stage/unstage`功能



## 存储卷的生命周期

![csi_lifecycle.png](.pics/extend_csi/csi_lifecycle.png)

## CSI 实现

**node.yaml**（必选）

- 通过 DaemonSet 在每个节点启动一个 CSI 插件，为 kubelet 提供 CSI Node 服务；

  - 包含 `node-driver-registrar`边车容器，向 kubelet 注册 CSI 插件（通过访问CSI Identity 服务获取插件信息）；

**controller.yaml**（可选）

- 通过 Deployment / StatefulSet 再启动一个插件，提供 CSI Controller 服务；

  - 根据所需的接口，包含`external-provisioner`等边车容器；

**driver.yaml**（必选）

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: testcsidriver.example.com
spec:
  # cause Kubernetes to skip attach(ControllerPublishVolume) operations for all CSI Driver testcsidriver.example.com volumes.
  # controller pod does not need 'external-attach' side car
  attachRequired: false
  # CSI driver requires additional pod information (like podName, podUID, etc.) during mount operations
  # in the NodePublishVolumeRequest.volume_context map
  podInfoOnMount: true
```



### 自定义的Driver Plugin容器

> ~~废弃：https://github.com/kubernetes-csi/drivers~~，提供了默认的 Identity, Node, Controller的类，不可用于生产。

自实现可以参考 [NFS Driver](https://github.com/kubernetes-csi/csi-driver-nfs)



### SideCar 容器

这些外部组件会作为`sidecar`容器和 CSI 插件放置在一个 Pod 中。

#### node-driver-registrar 

> **Git Repository:** https://github.com/kubernetes-csi/node-driver-registrar

负责请求 Identity Service 来获取插件信息并且注册到 kubelet，初始化的时候通过 csi-sock 的 RPC 获取 driver name 和 node id。主要功能给 node 打上 annotations，device-driver-registar 主要是注册 Node annotation。

#### external-provisioner

> **Git Repository:** https://github.com/kubernetes-csi/external-provisioner

针对 `Dynamic Provisioning`，根据 PVC 自动创建出 Volume

-  监听到 PVC创建，会调用 CSI driver Controller 服务的 `CreateVolume` 接口
-  组件监听到 PVC 的删除后，会调用 CSI driver Controller 服务的 `DeleteVolume` 接口

#### external-attacher

- 监听 VolumeAttachment 对象，并调用 CSI driver Controller 服务的 `ControllerPublishVolume` 和 `ControllerUnpublishVolume` 接口，用来将 volume 附着到 node 上，或从 node 上删除。

#### external-resizer

> https://github.com/kubernetes-csi/external-resizer

- watches Kubernetes PersistentVolumeClaims objects and triggers controller side expansion operation against a CSI endpoint

#### external-snapshotter

- watches Kubernetes Snapshot CRD objects and triggers CreateSnapshot/DeleteSnapshot against a CSI endpoint.

#### livenessprobe

> **Git Repository:** https://github.com/kubernetes-csi/livenessprobe

- monitors the health of the CSI driver and reports it to Kubernetes via the [Liveness Probe mechanism](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

#### external-health-monitor-controller

> **Git Repository:** https://github.com/kubernetes-csi/external-health-monitor

- deployed together with the CSI controller driver
- calls the CSI controller RPC `ListVolumes` or `ControllerGetVolume` to check the health condition of the CSI volumes and report events on `PersistentVolumeClaim` if the condition of a volume is `abnormal`.
- watches for node failure events (enable-node-watcher` flag to `true, only have effects on local PVs )



## 三方示例

[Rook](https://www.rook.io/)：基于Ceph的Kubernetes存储插件，加入水平扩展、迁移、灾难备份、监控等大量的企业级功能。

