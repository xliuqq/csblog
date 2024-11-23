# K8s安全机制

## Pod Security

> `hostPID` - Use the host’s pid namespace. Optional: Default to false.
>
> `hostIPC` - Use the host’s ipc namespace. Optional: Default to false.
>
> -  **container processes to communicate with processes on the host**

### SecurityContext 

三种级别的设置：

- **Container**级别：Container-level Security Context
- **单Pod**内所有容器和Volume：Pod-level Security Context
- **集群内部所有Pod和Volume**：Pod Security Policies

配置安全上下文可以

- 访问权限控制：指定容器中运行进程的用户和用户组（通过uid和gid）；
- 阻止容器使用 root 户运行（容器的默认运行用户通常在其镜像中指定，所以可能需要阻止容器 root 用户运行；
- 使用特权模式运行容器，使其对宿主节点的内核具有完全的访问权限；
- 与以上相反，通过添加或禁用内核功能，配置细粒度的内核访问权限；
- 设 SELinux C Security aced Linux 安全增强型 Linux ）边项，加强对容器的限制；
- 阻止进程写入容器的根文件系统

```bash
kubectl explain 'pods.spec.securityContext'
```

Pod的设置：

```yaml
apiVersion: v1 
kind: Pod 
metadata: 
  name: pod-as-user-guest 
spec: 
  containers: 
  - name: main
    image: alpine 
    command: ["/bin/sleep","99999"]
    securityContext: 
      # 特定用户，只能指定用户Id
      runAsUser: 405
      # 特定用户组
      runAsGroup: 1000
      #
      fsGroup:
      
      # 阻止容器使用 root 用户运行
      runAsNonRoot: true
```



## API Server 认证

K8s集群中所有资源访问和变更都是通过k8s API Server的REST API实现，

认证（Authentication）：识别客户端的身份；

### API 访问方式

K8s API的访问方式分类：

- 证书方式访问的普通用户或进程，包括运维人员、kubectl、kubelets等进程；
- Service Account方式访问的K8s的内部服务进程；
- 匿名方式访问的进程。

### API 认证方式

- HTTPS 证书认证：默认，基于**CA根证书签名的双向数字证书**认证方式；
- HTTP Bear Token认证：通过**Bearer Token**识别合法用户，指定存储的文件（存储token对应的用户信息，csv格式）；
- OpenID Connect Token认证：通过第三方OIDC协议进行认证；
- Webhook Token认证：通过**外部Webhook服务**进行认证；
- Authentication Proxy认证：通过认证代理程序进行认证；

## API Server 授权

### 授权策略

默认为`--authorization-mode=Node,RBAC`

- ~~AllowDeny：拒绝所有，仅用于测试；~~
- ~~AlwaysAllow：允许所有，集群不需要授权时使用；~~
- **ABAC**：基于属性的访问控制；
- **RBAC**：基于角色的访问控制；
- **Webhook**：基于外部的REST服务进行授权；
- **Node**：对kubelet进行授权的特殊模式；

### RBAC

Kubernetes提供了一系列机制以满足多用户的使用，包括**多用户，鉴权，命名空间，资源限制**等等。

RBAC是Kubernetes 默认进行**权限控制**的方式。用户与角色绑定，赋予角色权限；

#### User

- 在Kubernetes里**User只是一个用户身份辨识的ID**，没有真正用户管理；
- 通过**第三方提供用户管理和存储**，k8s通过User进行身份验证与权限认证；

- Kubernetes用户验证支持**X509证书认证，token认证和密码验证**几种方式；

`User`：字符串标识，通常应该在客户端CA证书中进行设置，K8s内置系统级别的用户/用户组，以"system:"开头；

#### Service Account

> 每个 Kubernetes 名字空间至少包含一个 ServiceAccount：也就是该名字空间的默认服务账号， 名为 `default`。
>
> - 在创建 Pod 时没有指定 ServiceAccount，自动将该名字空间中名为 `default` 的 ServiceAccount 分配给该 Pod；
> - `  automountServiceAccountToken: false`字段可以配置不自动设置

K8s内置，属于账号的一种，但是不是给K8s集群的用户（系统管理员、运维人员、租户用户等）使用，而是**给运行在Pod中的进程使用**，提供必要的身份证明。

- K8s 会为 `ServiceAccount`自动创建 并分配 Secret 对象（`token`, `ca.crt`, `namespace`三个数据）；
  - container 中的目录位于：`/var/run/secrets/kubernetes.io/serviceaccount/token`；

- Pod 可以在`spec.ServiceAccountName`中使用`ServiceAccount`，如果不指定则K8s为Pod分配默认的`ServiceAccount`；
  - **默认的`ServiceAccount`没有关联任何Role，；**
  - 生产环境，建议为所有`Namespace`下默认的`ServiceAccount`绑定只读权限的`Role`；

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: drone
  namespace: default
# 自动创建
secrets:
- name: drone-token-lrgmd
```

ServiceAccount 中的字段还有：

- `imagePullSecrets`：Pod 使用该 ServiceAccount 时，可以自动使用该字段进行镜像拉取；
- `secrets`：用于限制 Pod 只能访问那些密钥（仅当该 ServiceAccount 具有  `kubernetes.io/enforce-mountable-secrets` 的注解值为 true 时）；

#### Group

如果为K8s配置外部认证服务，则“用户组”的概念由外部认证服务提供；

- `ServiceAccount`对应的“用户”是：`system:serviceaccount:<NameSpace 名><ServiceAccount名>`，对应的内置“用户组”是`system:serviceaccounts:<Namespace名>`
- ``Group`：与用户名类似，通常应该在客户端CA证书中进行设置，不以"system:"为前缀；

#### Role

定义角色，受限于名空间。单个规则的字段：

- `apigroup`：API 组，比如`Pod/Deployment`等在`""`组下，`Job/CronJob`在`batch`组下；
- `resource`：对应API组下的资源，如 Pod等；
- `verbs`：允许的操作，如`get,list,watch,create,update,patch,delete`；
- `resourceName`：数据权限，限定可访问的资源名称； 空集合意味着允许所有资源。
  - 对`list`, `watch`, `create`, `deletecollection`操作无效


注：Role或ClusterRole与RoleBinding或ClusterRoleBinding**绑定之后，则Role/ClusterRole无法修改**；

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-role
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - '*'
  verbs:
  - '*'
```

#### RoleBinding

将用户和角色绑定，处于给定命名空间中的 **RoleBinding 仅在该命名空间中有效**。

- 可以引用相同命名空间中的 Role 或全局命名空间中的 ClusterRole；

**subjects** 字段：

- **subjects.kind** ：必需，被引用的对象的类别。 这个 API 组定义的值是 `User`、`Group` 和 `ServiceAccount`。 

- **subjects.name** ：必需被引用的对象的名称。
- **subjects.apiGroup**：apiGroup 包含被引用主体的 API 组。 对于 ServiceAccount 主体默认为 ""。 对于 User 和 Group 主体，默认为 "rbac.authorization.k8s.io"。
- **subjects.namespace**：被引用的对象的命名空间。 “User” 或 “Group” 不支持该字段。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: example-rolebinding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: drone
subjects:
- kind: User
  name: drone
  apigroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: drone
  namespace: default
# 应用于`default`名空间下的所有ServiceAccount
- kind: Group
  name: system:serviceaccounts:default
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRole

定义集群角色，不受名空间限制。

- K8s内置ClusterRole，包括admin，view，edit等

#### ClusterRoleBinding

将用户和集群角色绑定，不受限于名空间。



### 示例：创建权限限定的 kubeconfig

接下来将创建一个名为test的用户，其拥有test命名空间下的管理员权限，该命名空间有着CPU，内存，Pod数量等限制。

#### 创建ServiceAccount

创建`namespace/test`：**每创建一个命名空间，都会为其新建一个名为default的serviceaccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: starfish1
  namespace: ai-education
```

#### 创建Role

`kubectl apply -f role-ai-education.yaml`

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: ai-education
  name: starfish1-role
rules:
  - apiGroups: ["", "extensions", "apps"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["batch"]
    resources:
      - jobs
      - cronjobs
    verbs: ["*"]
```

#### 创建 RoleBinding

`kubectl apply -f rolebinding-ai-education.yaml`

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: starfish1-rolebinding
  namespace: ai-education
subjects:
- kind: ServiceAccount
  name: starfish1
  namespace: ai-education
roleRef:
  kind: Role
  name: starfish1-role
  apiGroup: rbac.authorization.k8s.io
```

#### 创建kubeconfig

 获取token相关信息，利用最后一行的secrets创建认证信息

```shell
kubectl get serviceaccount starfish1 -n ai-education -o yaml 
```

指定server、name（2.4.1节的token名）、namespace信息

```shell
server=https://172.16.2.132:6443
name=starfish1-token-m4bdx
namespace=ai-education

token=$(kubectl get secret/$name -n $namespace -o jsonpath='{.data.token}' | base64 --decode)
ca=$(kubectl get secret/$name -n $namespace -o jsonpath="{['data']['ca\.crt']}")

echo "apiVersion: v1
kind: Config
clusters:
- name: cluster.local
  cluster:
    certificate-authority-data: ${ca}
    server: ${server}
contexts:
- name: cluster.local
  context:
    cluster: cluster.local
    user: starfish1
current-context: cluster.local
users:
- name: starfish1
  user:
    token: ${token}
" > starfish1.kubeconfig
```

#### 使用KubeConfig

##### 可选1：指定kubeconfig路径

```shell
export KUBECONFIG=/path/starfish1.kubeconfig
```

##### 可选2：设置context

设置kubeconfig的user，其名称为test（<TOKEN_CONTENT>就是`serviceaccount/starfish1`的token）：

```shell
kubectl config set-credentials test --token=<TOKEN_CONTENT>
```

然后设置context，名称为`test-context`，引用user为test，cluster设置kuebconfig配置文件中环境选项中的集群，命名空间为test：

- 会在`~/.kube/config`中添加`test-context`的信息；

```shell
kubectl config set-context test-context --user=test --cluster=<Cluster Name> --namespace=test
```

最后，切换至该context：

```shell
kubectl config use-context test-context
```



## Admission Control

准入控制器的插件列表，支持用户自定义扩展。

- [**Mutating Admission Webhook，Validating Admission Webhook**](./k8s.md#Webhook)





# 多租户管理



## 多租户方案 HNC（K8s-builtin）

- 层级化的 Namespace 的结构；

  

## Operator实现用户管理

> 参考：[Kubernetes Operator实现用户管理](https://mp.weixin.qq.com/s/p7i5qn3ipvJXGQl-Y14EBQ)

- 在Kubernetes里创建User自定义资源，使用LDAP存储用户帐号信息；
- 通过Kubernets CertificateSigningRequest请求X509证书，生成Kubeconfig；
- 通过各种自定义Role资源来创建Kubernetes Role与用户绑定，分配用户权限；
- 最终用户通过客户端使用kubeconfig来访问Kubernetes资源。

**主要流程**

1. User控制器调谐，创建LDAP用户，创建用户KubeConfig的Configmap；

2. 在CreateKubeConfig生成kubeconfig用户信息，创建CertificateSigningRequest；

3. 在Informer中监听CertificateSigningRequest事件，Approve请求，更新Configmap中用户kubeconfig的证书。



## 轻量级多租户（KubeZoo-字节-非开源）

> [轻量级 Kubernetes 多租户方案的探索与实践](https://mp.weixin.qq.com/s/V6x9V4AW3-XFEmv_b8LmvA)

在云计算时代，就出现了多个租户共享同一个 Kubernetes  集群的需求。

在这方面，社区的`Kubernetes Multi-tenancy Working Group`定义了三种 Kubernetes 的多租户模型：

- `Namespaces as a Service`：多个租户共享一个 Kubernetes 集群，每个租户被限定在自己的 Namespace 
  - 只能使用 Namespace 级别的资源，**不能使用集群级别的资源**，它的 API 兼容性比较受限。
- `Clusters as a Service`或`Control planes as a Service`：租户间做物理集群隔离的方案。每个租户都有独立的 Master，通过 Cluster API 或 Virtual Cluster 等项目完成它的生命周期管理。
  - 每个租户都会有一套独立的控制面组件，包括 API Server、Controller Manager 以及自己的 Scheduler

![图片](.pics/k8s_security/zoo_compare.png)

**KubeZoo 目的：在一个K8s集群中，实现多租户隔离，租户可以创建集群级别的资源（如Namespace等）**

### 架构和原理

![图片](.pics/k8s_security/zoo_arch.png)

KubeZoo 作为一个**网关服务**，部署在 API Server 的前端。它会抓取所有来自租户的 API 请求，然后注入租户的相关信息，最后把请求转发给 API Server，同时也会处理 API Server 的响应，把响应再返回给租户。

KubeZoo 的核心功能是**对租户的请求进行协议转换**，使得每个租户看到的都是独占的 Kubernetes 集群。对于后端集群来说，多个租户实际上是利用了 Namespace 的原生隔离性机制而共享了同一个集群的资源。



限制：

- 针对 Daemonset 和 Node 等集群共享资源对象是受限制；