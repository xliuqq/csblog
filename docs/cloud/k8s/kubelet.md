# Kubelet

- 宿主机进程：注册到`/etc/systemd/system/kubelet.service`

- `syncFrequency`：默认`1min`，表示在运行中的容器与其配置之间执行同步操作的最长时间间隔；



### Static Pod

Kubernetes中有一种特殊的容器启动方法，称为`Static Pod`，允许把Pod的Yaml放在指定目录，当`kubelet`启动时，会自动检查该目录，加载所有的Pod YAML并启动。

kubeadm 创建的 K8s 集群，Master组件的 Yaml 会被生成在`/etc/kubernetes/manifests`路径下：

- `etcd.yaml`、`kube-apiserver.yaml`、`kube-controller-manager.yaml`、`kube-scheduler.yaml`；



## 常用命令

### 标记节点不可调度

```shell
$ kubectl cordon $NODENAME
```