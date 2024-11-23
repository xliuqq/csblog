# UI和管理

## Dashborad

![Kubernetes Dashboard UI](.pics/ui_manager/ui-dashboard.png)

### 部署

涉及到的国外镜像，如果无法访问，下载下来之后，改镜像地址。

```bash
# 添加在线 Helm Repo，或者下载 Helm 包 https://github.com/kubernetes/dashboard/releases
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

# helm upgrade --install kubernetes-dashboard kubernetes-dashboard-XX --create-namespace --namespace kubernetes-dashboard

# 查看
kubectl get pods -n kubernetes-dashboard
```



### 使用

默认情况下，Dashboard 会使用最少的 RBAC 配置进行部署。 

- 使用 ClusterIP，外部无法访问；
- 默认的 kubernetes-dashboard 的 ServiceAccount 的权限不够，界面数据没有权限。

```bash
# 配置代理，接受所有地址的访问
kubectl  proxy --address="172.16.2.131" --port=8001 --accept-hosts="^.*"

# 查看kubernetes-dashboard的token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep kubernetes-dashboard | awk '{print $1}')
```

kubernetes-dashboard 名空间下，创建 admin-user 的 Service Account，并绑定 cluster-admin 的角色。

- yaml 定义如下

```yaml
cat > dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard  
EOF
```

执行命令，创建角色，获取 token。

```bash
# 创建了一个叫 admin-user 的服务账号，并放在kubernetes-dashboard 命名空间下，并将 cluster-admin 角色绑定到admin-user账户，这样admin-user账户就有了管理员的权限。默认情况下，kubeadm创建集群时已经创建了cluster-admin角色，我们直接绑定即可。
kubectl apply -f dashboard-adminuser.yaml
# 查看admin-user账户的token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

```

## [KubeSphere](https://github.com/kubesphere/kubesphere)

> kubesphere 4.x 以后的版本[商用受限](https://github.com/kubesphere/kubesphere/blob/master/LICENSE)。

[KubeSphere](https://kubesphere.com.cn/) 是在 Kubernetes之上构建的面向云原生应用的 **容器混合云**，支持多云与多集群管理，提供全栈的 IT 自动化运维的能力，简化企业的 DevOps 工作流。

<table>
  <tr>
      <td width="50%" align="center"><b>工作台</b></td>
      <td width="50%" align="center"><b>项目资源</b></td>
  </tr>
  <tr>
     <td><img src=".pics/ui_manager/console.png"/></td>
     <td><img src=".pics/ui_manager/project.png"/></td>
  </tr>
  <tr>
      <td width="50%" align="center"><b>CI/CD 流水线</b></td>
      <td width="50%" align="center"><b>应用商店</b></td>
  </tr>
  <tr>
     <td><img src=".pics/ui_manager/cicd.png"/></td>
     <td><img src=".pics/ui_manager/app-store.png"/></td>
  </tr>
</table>

### 架构

<img src=".pics/ui_manager/kubesphere-architecture.png" alt="Architecture" style="zoom: 50%;" />

### 部署使用

[快速入门](https://github.com/kubesphere/kubesphere/blob/master/README_zh.md#快速入门)



## Rancher

Rancher是一个开源的容器管理平台，为在生产中部署容器的组织构建。Rancher可以轻松地在任何地方运行Kubernetes，满足IT需求，并为DevOps团队提供支持。

### 架构

<img src=".pics/ui_manager/rancher-architecture-rancher-api-server.svg" alt="Architecture" style="zoom:67%;" />

### 安装

[高可用安装指南 | Rancher文档](https://docs.rancher.cn/docs/rancher2/installation/install-rancher-on-k8s/_index)