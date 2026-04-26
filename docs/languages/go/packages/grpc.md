# Grpc



## 示例(TODO，给出 github 链接)

`product.proto`

```protobuf
syntax = 'proto3'
package ecommerce.v1;

message ProductID {
	string value = 1;
}
message Product {
	string id = 1;
	string name = 2;
	string description = 3;
	float price = 4;
}

service ProductInfo {
	rpc addProduct(Product) returns (ProductID);
	rpc getProduct(ProductID) returns (Product);
}
```

`go.mod`

```go
module productinfo/service

require (
    github.com/golang/protobuf v1.3.2
    google.golang.org/grpc v1.24.0
)
```

安装编译插件

```shell
go get -u google.golang.org/grpc
go get - github.com/golang/protobuf/protoc-gen-go

protoc -I ecommerce ecommerce/product.proto --go_out=plugins=grpc:<module_dir_path>/ecommerce
```



**grpc 服务端**

```go
func main() {
    lis, err = net.Listen("tcp", port)
    ...
}
```



```shell
go build -i -v -o bin/server
```

