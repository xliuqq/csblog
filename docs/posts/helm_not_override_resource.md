---
date: 2023-11-05
readtime: 5
categories:
  - helm
---



# Helm 如何升级时不升级特定的Resource

Helm 3的应用在升级时，会根据[三路合并](https://helm.sh/docs/faq/changes_since_helm2/#improved-upgrade-strategy-3-way-strategic-merge-patches)策略去决定如何对 Resource 进行升级。

假设某个 Helm Applicaion 定义了 ConfigMap，在安装的时候会在 K8s 创建对应资源，但是后续运维人员会根据生产环境情况去动态修改该 ConfigMap 中的内容，并且希望在 Application 升级的时候，对该 ConfigMap 不进行升级（即不能修改生产环境的配置内容），该如何配置 Helm Application 的 Charts 内容呢？



<!--- more --->

## Helm 2 two-way 合并策略

Helm 2 在升级过程中，会对比最近一次的chart manifest 和 `helm upgrade`提供的 chart manifests。升级会对比**两个 chart 不同**来决定哪些更改会应用到Kubernetes资源中。

- 如果 resource 被非 helm 的外部更改（如 kubectl edit），这些变更不会被考虑（helm 不会对这些变更做升级或回滚）；
- 即：**只会根据前后提供的 helm charts 进行对比，将不同之处 patch 到 resource**；



在 Helm 2 的 场景下，只要不同版本的 Helm Charts 对应的 ConfigMap 的 Resource 内容不变，则通过 kubectl edit 的修改，不会被 Helm 升级而内容被重置，满足需求场景。

## Helm 3 three-way 合并策略

考虑 old manifest，live state 和 new manifest 三者，生成 patch。

- **如果 manifest 前后不变，helm 3 会将新的 mainfest 覆盖，此时外部的改动会丢失**；



## Helm 3 配置升级时忽略特定Resource

> Github链接：[Feature request: ignore part of a resource in upgrade](https://github.com/helm/helm/issues/7971#issuecomment-1147885486)

官网没有配置可以解决该问题，通过Github Issues 搜索，可以通过 `lookup` 函数解决

- helm 3.13 版本的 `--dry-run`（进查看变更而不应用到集群） 参数支持`client`和`server`（支持 `lookup`）模式

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: upgrade-test
  namespace: default
data:
  content: |
# 查找 config map 的当前内容，并应用  
{{ (lookup "v1" "ConfigMap" "default"  "upgrade-test").data.content | indent 4 }}

```

