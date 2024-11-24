# Habor

Harbor 是为企业用户设计的容器镜像仓库开源项目，包括了权限管理(RBAC)、LDAP、审计、安全漏洞扫描、 镜像验真、管理界面、自我注册、HA 等企业必需的功能，同时针对中国用户的特点，设计镜像复制和中文支持等功能。

- use Harbor to **proxy and cache** images from a target **public or private registry**

## 组件

**Nginx(Proxy)**：用于代理Harbor的registry,UI, token等服务 

**db**：负责储存用户权限、审计日志、Dockerimage分组信息等数据

**UI**：提供图形化界面，帮助用户管理registry上的镜像, 并对用户进行授权 

**jobsevice**：负责镜像复制工作的，他和registry通信，从一个registry pull镜像然后push到另一个registry，并记录job_log 

**Adminserver**：是系统的配置管理中心附带检查存储用量，ui 和 jobserver启动时候回需要加载 adminserver 的配置

**Registry**：原生的docker镜像仓库，负责存储镜像文件

**Log**：为了帮助监控Harbor运行，负责收集其他组件的log，记录到syslog中

<img src=".pics/habor/habor.png" alt="img" style="zoom: 33%;" />

## Kubernetes 拉取Harbor仓库私有项目镜像

> docker(moby) 没有类似 yum / npm 这种全局配置代理源的方式。
>
> - https://github.com/moby/moby/pull/34319

image 配置 harbor 的地址

```shell
# docker 登录，拉去私有镜像
docker login -u 用户名 -p 密码  $Habor_Address

cat ~/.docker/config.json |base64 -w 0
```

k8s 通过创建  Secret 使用私有 Habor 镜像

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harborlogin
type: kubernetes.io/dockerconfigjson
data:
  # 上面 base64 加密的内容
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSIxMjcuMC4wLjEiOiB7CgkJCSJhdXRoIjo...
```

yaml 文件中注明 imagePullSecrets 属性



### 构建指定命名空间的Harbor仓库密钥

```shell
kubectl create secret docker-registry hc-secret --docker-server=172.16.2.132:30002 --docker-username=hc --docker-password=Hc123456 -n ai-education
```

- docker-registry：密钥类型
- hc-secret：密钥名称
- --docker-server：镜像仓库地址
- --docker-username：镜像仓库用户名
- --docker-password：镜像仓库密码
- -n：需拉取镜像对应服务的命名空间

### yaml部署时使用对应密钥

```yaml
spec:
  imagePullSecrets:
  - name: hc-secret
```

