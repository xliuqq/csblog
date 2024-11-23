# KubeVela

> [KubeVela 简介 | KubeVela](https://kubevela.io/zh/docs/)

开箱即用的现代化**应用交付与管理平台**，它使得应用在面向混合云环境中的交付更简单、快捷。

- 通过 [开放应用模型（OAM）](https://oam.dev/)来作为应用交付的顶层抽象；



![what-is-kubevela](.pics/kubevela/what-is-kubevela.jpg)

## OAM

### 应用（Application）

定义了一个微服务业务单元所包括的制品（二进制、Docker 镜像、Helm Chart...），**由组件（Component）、运维特征（Trait）、工作流（Workflow）、应用策略（Policy）四部分组成。**



## 示例

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: first-vela-app
spec:
  components:
    - name: express-server
      type: webservice
      properties:
        image: oamdev/hello-world
        ports:
         - port: 8000
           expose: true
      traits:
        - type: scaler
          properties:
            replicas: 1
  policies:
    - name: target-default
      type: topology
      properties:
        # local 集群即 Kubevela 所在的集群
        clusters: ["local"]
        namespace: "default"
    - name: target-prod
      type: topology
      properties:
        clusters: ["local"]
        # 此命名空间需要在应用部署前完成创建
        namespace: "prod"
    - name: deploy-ha
      type: override
      properties:
        components:
          - type: webservice
            traits:
              - type: scaler
                properties:
                  replicas: 2
  workflow:
    steps:
      - name: deploy2default
        type: deploy
        properties:
          policies: ["target-default"]
      - name: manual-approval
        type: suspend
      - name: deploy2prod
        type: deploy
        properties:
          policies: ["target-prod", "deploy-ha"]
```

