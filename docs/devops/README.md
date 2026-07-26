# DevOps



## GitOps

> 一般用于 K8s 集群的 CD 场景，保证集群中最新版本的部署。

以**声明方式管理其Kubernetes部署**，**定期轮询存储库来将存储在源代码存储库**中的`Kubernetes manifests文件`与`Kubernetes`集群同步。

### 传统的Jenkins + K8s

#### 流程

- 开发人员创建代码并编写Dockerfile。他们还为应用程序创建Kubernetes manifests和Helm Charts。
- 他们将代码推送到源代码存储库。
- 源代码存储库使用提交后的钩子触发Jenkins构建。
- Jenkins CI流程将构建Docker映像和Helm软件包，并将其推送到依赖仓库。
- 然后，Jenkins CD程序部署helm charts到k8s cluster。

#### 问题

- Kubernetes **凭据存储**在Jenkins服务器中；
- 可以使用 Jenkins 创建和更改配置，但**无法使用它删除现有资源**；

### Gitops CD

- 资源配置文件清单：Git Repo 中；
- 源拉取组件：webhook 通知 / 定时轮询；
- 资源同步控制器：对比期望状态（清单定义）vs 集群实时状态，执行增删改；

#### [FluxCD](https://github.com/fluxcd/flux2)

![notification-controller-overview.png](./.pics/README/notification-controller-overview.png)

`Notification Controller`通过对 `Source`（GitRepository）的修改（注解），触发实时更新。

- `Source Controller` 配置 Git 仓库轮询间隔（兜底）;

集群内只由 source-controller 统一访问外部 Git 仓库，减少对外连接，下游控制器从 source-controller 下载Git 仓库代码形成的 Artifact 压缩包。

- kustomize-controller **定时执行一次完整协调（集群自愈漂移）**，同时监听`GitRepository.status.artifact` 发生变更，立即协调；

#### Argo CD

> Argo CD 提供UI界面，以及命令行操作，支持多套集群（不同 kubeconfig 配置）。

![argo cd](./.pics/README/argocd.png)

***Repository Server***：an internal service which maintains **a local cache of the Git repository** holding the application manifests；

- 负责拉取渲染；

***Application Controller***：Kubernetes controller which continuously monitors running applications and compares the current, live state against the desired target state

- 定义源清单仓库地址/路径及目标集群/名空间；