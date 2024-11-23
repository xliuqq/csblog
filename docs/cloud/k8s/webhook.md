# Webhook

>  Webhook 默认按照**字母序**进行各个 Webhook 的调用。

k8s apiserver 在处理每一个操作资源对象的请求时，在经过认证（是否为合法用户）/ 鉴权（用户是否拥有权限）后，并不会直接根据端点资源类型和 rest 动作直接执行操作，在这中间，请求会被一系列准入控制器插件（Admission Controller）进行拦截，校验其是否合乎要求，亦或是对特定资源进行配置。



## 准入机制

- 如果所有的 webhooks 批准请求，准入控制链继续流转；
- 如果有任意一个 webhooks 阻止请求，那么准入控制请求终止，并返回第一个 webhook 阻止的原因。其中，多个 webhooks 阻止也只会返回第一个 webhook 阻止的原因；
- 如果在调用 webhook 过程中发生错误，那么请求会被终止或者忽略 webhook



## 准入控制器类型

1. **变更** (**Mutating**)：解析请求并在请求向下发送之前对请求进行**更改**；第一阶段
2. **验证** (**Validating**)：解析请求并根据特定数据进行验证；第二阶段



## 动态准入控制

以 Webhook 的形式去编写自定义的准入控制器，对特定资源的请求进行定制修改和验证，典型的场景有通过 mutating 类型 Webhook 注入 side-car 到 pod。

以 kubebuilder 脚手架为例，实现`Handler`接口`Handle(context.Context, Request) Response`

```go
import (
	"github.com/go-logr/logr"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
)

// 定义webhook的path和处理的资源，针对create和update的pods

// +kubebuilder:webhook:path=/mutate-fluid-io-v1alpha1-schedulepod,mutating=true,failurePolicy=fail,sideEffects=None,admissionReviewVersions=v1;v1beta1,groups="",resources=pods,verbs=create;update,versions=v1,name=schedulepod.fluid.io
var (
	// HandlerMap contains admission webhook handlers
	HandlerMap = map[string]common.AdmissionHandler{
		common.WebhookSchedulePodPath: &CreateUpdatePodForSchedulingHandler{},
	}
)
	
func Register(mgr manager.Manager, client client.Client, log logr.Logger) {
	server := mgr.GetWebhookServer()
	for path, handler := range HandlerMap {
		handler.Setup(client)
        // 注册 handler
		server.Register(path, &webhook.Admission{Handler: handler})
		log.Info("Registered webhook handler", "path", path)
	}
}
```



### MutatingWebhookConfiguration

> Webhook 默认按照字母序进行各个 Webhook 的调用，为了保证 Webhook 能够在最后一个调用，设置 `reinvocationPolicy: IfNeeded`（在 Webhook 阶段，如果在调用后其被其他 Webhook 修改，则再次调用该 Webhook）即可。

示例

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: fluid-pod-admission-webhook
webhooks:
  - name: schedulepod.fluid.io
    rules:
      - apiGroups:   [""]
        apiVersions: ["v1"]
        operations:  ["CREATE","UPDATE"]
        resources:   ["pods"]
    clientConfig:
      service:
        namespace: fluid-system
        name: fluid-pod-admission-webhook
        path: "/mutate-fluid-io-v1alpha1-schedulepod"
        port: 9443
      caBundle: Cg==
    timeoutSeconds: 20
    failurePolicy: Fail
    sideEffects: None   # 说明此 Webhook 是否有副作用
    admissionReviewVersions: ["v1","v1beta1"]
    reinvocationPolicy: Never #在一次录取评估中，Webhook 被调用的次数不会超过一次
    namespaceSelector:
      matchLabels:
        fluid.io/enable-injection: "true"
```

