# Go Test

## Run

`go run main.go`

- go run main.go时，不会自动加载main.go同级目录下，同一个package不同文件

但是，如果main.go文件里面调用了，同级目录下(同一个package不同文件)test.go文件的函数，则需要通过以下语句调用：

- `go run .`    run之后的点，代表当前目录
- `go run main.go test.go`   直接使用文件路径，来手动加载引入文件



## 原生Go Test

在测试函数中，使用Error，Fail或者相关方法来表示测试失败。

- 测试文件与源代码文件放在一块，测试文件的文件名以`_test.go`结尾。

- 测试文件不会被正常编译，只会在使用`go test`命令时编译。



测试用例有四种形式：

- TestXxxx(t *testing.T) // 基本测试用例
- BenchmarkXxxx(b *testing.B) // 压力测试的测试用例
- Example_Xxx() // 测试控制台输出的例子
- TestMain(m *testing.M) // 测试Main函数



### 子测试

`T`和`B`的`Run()`方法可以直接执行子功能测试和子性能测试，而不用为每一个测试用例单独编写一个测试函数。这样便可以使用类似于表驱动基准测试和创建分层测试。这种方法也提供了一种方式来处理共同的<setup code>和<tear-down code>。

```go
func TestFoo(t *testing.T) {
   // <setup code>
   t.Run("A=1", func(t *testing.T) { ... })
   t.Run("A=2", func(t *testing.T) { ... })
   t.Run("B=1", func(t *testing.T) { ... })
   t.Run("A=1", func(t *testing.T) { ... })
   // <tear-down code>
}
```

每一个子功能测试和子性能测试都有唯一的名字：顶层的测试函数的名字和传入`Run()`方法的字符串用`/`连接，后面再加上一个可选的用于消除歧义的字符串。上面的四个子测试的唯一名字是：

- **TestFoo/A=1**
- **TestFoo/A=2**
- **TestFoo/B=1**
- **TestFoo/A=1#01**

命令行中的`-run`参数和`-bench`参数是可选的，可用来匹配测试用例的名字。对于包含多个斜杠分隔元素的测试，例如subtests，参数本身是斜杠分隔的，表达式依次匹配每个name元素。由于是可选参数，因此空表达式匹配所有的字符串。

```go
go test -run ''      # Run all tests.
go test -run Foo     # Run top-level tests matching "Foo", such as "TestFooBar".
go test -run Foo/A=  # For top-level tests matching "Foo", run subtests matching "A=".
go test -run /A=1    # For all top-level tests, run subtests matching "A=1".
```



### 跳过函数

功能测试或性能测试时可以跳过一些测试函数。

```go
func TestTimeConsuming(t *testing.T) {
   if testing.Short() {
      t.Skip("skipping test in short mode.")
   }
   ...
}
```



## Ginkgo

*Ginkgo*是一个BDD风格的Go测试框架
