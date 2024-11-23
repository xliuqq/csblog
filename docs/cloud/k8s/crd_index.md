# [K8s CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

`CustomResourceDefinition`：K8s 的 CRD 的 Yaml 定义，kubebuilder 等三方软件最终也会产生这样的 Yaml 文件定义。

kubebuilder 和 operator sdk 都是编写 K8s Operator的工具，operator sdk 对于使用helm 更适配，operator底层基于kubebuilder开发。

- GO 开发 CRD：[kubebuilder](./crd_kubebuilder.md)
- Java 开发 CRD：[JavaOperator](./crd_java.md)