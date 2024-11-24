# Istio

## 架构

![istio_arch](.pics/istio/istio_arch.png)

### 数据平面

Istio的数据平面主要包括Envoy代理的扩展版本。 **Envoy是一个开源边缘和服务代理**，可帮助将网络问题与底层应用程序分离开来。应用程序仅向localhost发送消息或从localhost接收消息，而无需了解网络拓扑。

- 在OSI模型的**L3和L4层运行**的网络代理；
- 支持用于基于HTTP的流量的附加L7层过滤器；
- HTTP/2和gRPC传输具有一流的支持；



### 控制面

控制平面负责**管理和配置数据平面中的Envoy代理**。

在Istio架构中，控制面核心组件是**istiod**，Istiod负责**将高级路由规则和流量控制行为转换为特定于Envoy的配置**，并在**运行时将其传播到Sidecar**。



## 组件

### Envoy

Istio作为服务网格提供的许多功能实际上是由**Envoy代理**的基础内置功能启用的：

- **流量控制**：Envoy通过HTTP，gRPC，WebSocket和TCP流量的丰富路由规则启用细粒度的流量控制应用，如负载均衡，百分比流量分割等；
- **网络弹性**：Envoy包括对**自动重试，断路和故障注入**的开箱即用支持
- **安全性**：Envoy还可以实施安全策略，并对基础服务之间的通信应用访问控制和速率限制
- 基于 WebAssembly 的可插拔扩展模型，允许通过自定义策略执行和生成网格流量的遥测。

### Istiod

组件 Istiod 提供服务发现、配置和证书管理。

Istio 可以支持发现多种环境，如 Kubernetes 或 VM。



## 安装



```shell
# 下载
wget https://github.com/istio/istio/releases/download/1.16.1/istio-1.16.1-linux-amd64.tar.gz

tar -xzf istioctl-1.16.1-linux-amd64.tar.gz

cp istioctl-1.16.1/bin/istioctl /usr/local/bin/istioctl

# 安装
istioctl install --set profile=demo -y

# 查看 istio-system 命名空间下面的 Pod 运行状态
kubectl get pods -n istio-system
```



## 使用

### sidecar注入

自动注入：名空间允许自动注入sidecar

`kubectl label namespace default istio-injection=enabled`

手动注入：输出注入sidecar之后的 yaml

```shell
istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### 网关

在集群外部访问，就需要添加一个 istio gateway，gateway 相当于 k8s 的 ingress controller 和 ingress。



### 仪表盘





## 配置

### VirtualService

> 在 Istio 中定义路由规则，控制流量路由到服务上的各种行为。

示例：定义一个虚拟服务，根据请求是否来自某个特定用户，把它们路由到服务的不同版本去。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # 列出VirtualService的hosts，可以是IP、DNS名称、FQDN或*
  - reviews
  http: # 在下面配置VirtualService的路由规则，指定符合哪些规则的流量打到哪些Destination，支持HTTP/1.1，HTTP2，及gRPC等协议
  - match: # 指定具体的匹配规则
    - headers:
        end-user:
          exact: jason
    route:
    - destination: # 指定满足规则后将流量打到哪个具体的Destination
        host: reviews  # Kubernetes Service 短名称
        subset: v2
  - route:  # 流量规则按从上到下的优先级去匹配，若不满足上述规则时，进入该默认规则
    - destination:
        host: reviews
        subset: v3
```



### DestinationRule

> 定义将流量如何路由到指定目标地址，然后使用目标规则来配置该目标的流量。

将虚拟服务看成将流量如何路由到指定目标地址，然后使用目标规则来配置该目标的流量。目标规则将应用于流量的“真实”目标地址。

使用**目标规则来指定命名的服务子集**，例如按版本为所有指定服务的实例分组，然后可以在虚拟服务的路由规则中使用这些服务子集来控制到服务不同实例的流量。

示例：目标规则为 `my-svc` 目标服务配置了 3 个具有不同负载均衡策略的子集。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM  # 随机的策略
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN  # 轮询
  - name: v3
    labels:
      version: v3
```

