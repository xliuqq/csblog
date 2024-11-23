# 命令及使用

## kubectl

### apply

部署或者更新：`kubectl apply -f app.yaml`

### get

> json 路径匹配：
>
> - `-o jsonpath='{.items[?(@.metadata.uid=="43504a29-8b72-4304-873c-3fee910374fa")].metadata.name}'`

`–show-labels `: 查看标签

- 查看pod服务：`kubectl get pod -n cbc-dev | grep router`
- 查询服务pod 和所在的节点：`kubectl -n cbc-dev get pod -o wide`
- 查看deployment：`kubectl get deployments`
- 查看replicaset：`kubectl get rs`
- 查看namespace：`kubectl get ns` 


### describe

描述Pod信息：`kubectl -n cbc-dev describe pod podName `

### logs

查看pod 启动日志：`kubectl -n cbc-dev logs podName `

### exec

进入pod查看：`kubectl -n cbc-dev exec -it podName bash`

### expose

将资源暴露为service对外访问，支持： pod（po），service（svc），replication controller（rc），deployment（deploy），replica set（rs）

`kubectl expose  pod hc-base-jupyter-pod --port=8888 --target-port=8888  --type=NodePort -n ai-education`

- `--port`: 集群内部服务访问端口，即通过 clusterip + port 在容器内访问；
- `--target-port`：容器内的服务端口
- `--type`：服务暴露类型，一般选择NodePort，会**自动分配一个外部访问的port**（通过`kubectl get service`查看）
- `--name`：名称（可选）

### port-forward

通过**端口转发映射本地端口到指定的应用（Pod）端口**：`kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn`

### delete

删除pods：`kubectl -n cbc-dev delete -f app.yaml `

### label

节点打标签：`kubectl label nodes kube-node label_name=label_value`

删除标签（最后指定Label的key名并与一个减号相连）：`kubectl label nodes 194.246.9.5 gpu-`

### scale

Pod伸缩：`kubectl scale deployment nginx-deployment --replicas=4`

### edit

编辑Yaml配置：`kubectl edit deployment/nginx-deployment`

### rollout

回滚版本：`kubectl rollout undo deployment/nginx-deployment`

查看历史：`kubectl rollout history deployment/nginx-deployment`

查看滚动更新的状态：`kubectl rollout status deployment/nginx-deployment`

暂停（批量改配置)，暂停 -> 修改配置 -> 恢复 -> 滚动更新：`kubectl rollout pause deployment/nginx-deployment`

恢复：`kubectl rollout resume deployment/nginx-deployment`



## 容器内部访问 ApiServer

### 官方客户端库

默认的官方客户端，如 [Go 客户端库](https://github.com/kubernetes/client-go/)，通过`rest.InClusterConfig()`自动处理 API Server 的主机发现和身份认证

- 通过 ServiceAccount 的挂载信息进行认证，`/var/run/secrets/kubernetes.io/serviceaccount/`目录下
  - `token`，`ca.crt`，`namespace`



### RestAPI直接访问

两种方式获取 REST 地址：

- **`KUBERNETES_SERVICE_HOST` 和 `KUBERNETES_SERVICE_PORT_HTTPS` 环境变量**为 Kubernetes API 服务器生成一个 HTTPS URL。
- API 服务器的集群内地址也**发布到 `default` 命名空间中名为 `kubernetes` 的 Service** 中（ **`kubernetes.default.svc`** ）。

```shell
# 指向内部 API 服务器的主机名
APISERVER=https://kubernetes.default.svc
# 服务账号令牌的路径
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
# 读取 Pod 的名字空间
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
# 读取服务账号的持有者令牌
TOKEN=$(cat ${SERVICEACCOUNT}/token)
# 引用内部证书机构（CA）
CACERT=${SERVICEACCOUNT}/ca.crt

# 使用令牌访问 API
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api
```





