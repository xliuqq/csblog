# Go 版本管理

## GOPATH

Go 1.5 之前，Go 最原始的依赖管理使用的是 go get，执行命令后会拉取代码放入 `GOPATH/src`

- 无法进行版本控制，依赖冲突；

## Go Vendor

Go 1.5 版本推出了 vendor 机制，从 Go1.6 起，默认开启 vendor 机制。

- 每个项目的根目录下可以有一个 vendor 目录，里面存放了该项目的依赖的 package

## Go Module

Go 1.12 版本 Go Modules 成为默认的依赖管理方式

- **go.mod** 文件定义 module path 和依赖库的版本
- **go.sum** 的文件，该文件包含特定依赖包的版本内容的散列哈希值



### 示例

```makefile
# 如果你的版本已经 >=2.0.0，按照 Go 的规范，你应该加上 major 的后缀，如 
# module xliu.sample.com/clientgo/v2
# 好处是一个项目中可以使用依赖库的不同的 major 版本，它们可以共存
module xliu.sample.com/clientgo

# 代码所需要的 Go 的最低版本
go 1.18

# 需要的各个依赖库以及它们的版本
require (
	k8s.io/apimachinery v0.24.17
	k8s.io/client-go v0.24.17
)

# indirect 注释，间接的使用了这个库，但是又没有被列到某个 go.mod 
require (
	github.com/PuerkitoBio/purell v1.1.1 // indirect
	# 伪版本号，因为依赖库没有发布版本
	github.com/PuerkitoBio/urlesc v0.0.0-20170810143723-de5bf2ad4578 // indirect
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/emicklei/go-restful v2.9.5+incompatible // indirect
	github.com/go-logr/logr v1.2.0 // indirect
	github.com/go-openapi/jsonpointer v0.19.5 // indirect
	github.com/go-openapi/jsonreference v0.19.5 // indirect
	github.com/go-openapi/swag v0.19.14 // indirect
	github.com/gogo/protobuf v1.3.2 // indirect
	github.com/golang/protobuf v1.5.2 // indirect
	github.com/google/gnostic v0.5.7-v3refs // indirect
	github.com/google/gofuzz v1.1.0 // indirect
	github.com/imdario/mergo v0.3.5 // indirect
	github.com/josharian/intern v1.0.0 // indirect
	github.com/json-iterator/go v1.1.12 // indirect
	github.com/mailru/easyjson v0.7.6 // indirect
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
	github.com/modern-go/reflect2 v1.0.2 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/spf13/pflag v1.0.5 // indirect
	golang.org/x/net v0.8.0 // indirect
	golang.org/x/oauth2 v0.0.0-20211104180415-d3ed0bb246c8 // indirect
	golang.org/x/sys v0.6.0 // indirect
	golang.org/x/term v0.6.0 // indirect
	golang.org/x/text v0.8.0 // indirect
	golang.org/x/time v0.0.0-20220210224613-90d013bbcef8 // indirect
	google.golang.org/appengine v1.6.7 // indirect
	google.golang.org/protobuf v1.27.1 // indirect
	gopkg.in/inf.v0 v0.9.1 // indirect
	gopkg.in/tomb.v1 v1.0.0-20141024135613-dd632973f1e7 // indirect
	gopkg.in/yaml.v2 v2.4.0 // indirect
	gopkg.in/yaml.v3 v3.0.1 // indirect
	k8s.io/api v0.24.17 // indirect
	k8s.io/klog/v2 v2.60.1 // indirect
	k8s.io/kube-openapi v0.0.0-20220328201542-3ee0da9b0b42 // indirect
	k8s.io/utils v0.0.0-20220210201930-3a6ce19ff2f9 // indirect
	sigs.k8s.io/json v0.0.0-20211208200746-9f7c6b3444d2 // indirect
	sigs.k8s.io/structured-merge-diff/v4 v4.2.3 // indirect
	sigs.k8s.io/yaml v1.2.0 // indirect
)

# replace 可以替换某个库的所有版本到另一个库的特定版本，也可以替换某个库的特定版本到另一个库的特定版本。
```

