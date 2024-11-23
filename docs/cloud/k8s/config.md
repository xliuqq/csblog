# K8s中常用的配置

## kubeadm

在宿主机上运行`kubelet`，使用容器部署其它组件。

kubeadm生成的证书文件在Master节点的`/etc/kubernetes/pki`目录下，最主要的证书文件是`ca.crt`和对应的私钥`ca.key`。

Master组件的YAML文件会被生成在`/etc/kubernetes/manifests`路径下；



## ApiServer

### 修改NodePort范围

>  默认范围是 30000-32767。

使用 kubeadm 安装 K8S 集群的情况下，修改`/etc/kubernetes/manifests/kube-apiserver.yaml`文件，向其中添加 `--service-node-port-range=20000-22767` （定义需要的端口范围）

重启 `kube-apiserver`

```bash
# 获得 apiserver 的 pod 名字
export apiserver_pods=$(kubectl get pods --selector=component=kube-apiserver -n kube-system --output=jsonpath={.items..metadata.name})
# 删除 apiserver 的 pod，会自动创建新的 apiserver pod
kubectl delete pod $apiserver_pods -n kube-system
```

验证结果

- 执行以下命令查看相关pod

```bash
kubectl describe pod $apiserver_pods -n kube-system
```

注意：

- 对于已经创建的NodePort类型的Service，需要删除重新创建（TODO：端口前后范围冲突的处理？）；
- 如果集群有多个 Master 节点，需要逐个修改每个节点上的 `/etc/kubernetes/manifests/kube-apiserver.yaml` 文件，并重启 apiserver。



## 网络

### CoreDNS 添加域名映射

`kubectl edit configmap coredns -n kube-system`

- 在 `hosts` 中 添加主机名到 ip 的映射；

```yaml
data:
  Corefile: |
    .:53 {
    	... 
        hosts {
          172.16.1.129 apiserver.cluster.local
    	  # 添加主机名到 ip 的映射
          172.16.2.135   node135
          172.16.2.136   node136
          172.16.2.137   node137
          fallthrough
        }
        
        ...
        
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
    }
    # 特定DNS服务器解析特定域名
    *.baidu.com:53 {
        errors
        cache 30
        forward . 192.168.8.132
    }
```

