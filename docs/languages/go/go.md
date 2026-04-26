# Go基础语法

## 使用

go包下载的代理 https://goproxy.io/zh/

```shell
export GOPROXY=https://proxy.golang.com.cn,direct
```



## 变量

变量定义只有var，函数外不能用 := 赋值，变量未初始化有零值；

```go
var a int = 1  // 等价于 
var a  =       // 类型自动推断，等价于
a := 1
```

常量用const，不能用`:=`

```go
const a = 1
```

make也是一个内建函数，用来为 slice，map 或 chan 类型分配内存和初始化一个对象。

- new 的作用是初始化一个指向类型的指针(*T)，make 的作用是只为 slice，map 或 chan 初始化并返回引用（T）；

## 循环

```go
for key, value := range oldMap {
    newMap[key] = value
}
```





## 指针

**函数返回局部变量的地址是安全的**（但是C/C++中，函数返回局部变量的地址是不安全的）

- go语言编译器会**自动决定把一个变量放在栈还是放在堆**，编译器会做逃逸分析(escape analysis)，当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆；
- 对于**动态new出来的局部变量**，go语言编译器也会根据是否有逃逸行为来决定是分配在堆还是栈，而**不是直接分配在堆中**。



## 赋值

golang中只有值传递，没有引用传递。因为拷贝的内容有时候是非引用类型（int、string、struct等这些），这样就在函数中就无法修改原内容数据；有的是引用类型（指针、map、slice、chan等这些），这样就可以修改原内容数据。

### 值类型

值类型包括基本数据类型：**int**，**float**，**bool**，**string**，以及数组和结构体(**struct**)。

值类型变量声明后，不管是否已经赋值，编译器为其分配内存，此时该值存储于栈上。

```go
//数组的赋值
var c =[]int32{1,2,3}    //定义一个长度为3的int类型的数组
d := c      //将数组c赋值给d
d[1] = 100  //修改数组d中索引为1的值为100
fmt.Printf("c的值是%v，c的内存地址是%p\n",c,&c)   //c的值是[1 2 3]，c的内存地址是0xc42000a180
fmt.Printf("d的值是%v，d的内存地址是%p\n",d,&d)   //d的值是[1 100 3]，d的内存地址是0xc42000a1a0

```

自定义的struct

```go
func main() {
	p:=Person{"张三"}
	fmt.Printf("原始Person的内存地址是：%p\n",&p)
	modify(p)
	fmt.Println(p)
}

type Person struct {
	Name string
}

 func modify(p Person) {
	 fmt.Printf("函数里接收到Person的内存地址是：%p\n",&p)
	 p.Name = "李四"
 }

// 原始Person的内存地址是：0xc4200721b0 
// 函数里接收到Person的内存地址是：0xc4200721c0
// {张三}
```



### 引用类型

引用类型包括指针，**slice切片，map ，chan，interface**。

变量直接存放的就是一个内存地址值，这个地址值指向的空间存的才是值。所以修改其中一个，另外一个也会修改（同一个内存地址）。



## 枚举

`go` 中没有枚举变量，但是可以通过 `const` 实现同样功能。

```go
type WeekDay int 
const (
    SUNDAY WeekDay = iota  
    MONDAY 
    TUESDAY 
    WEDNESDAY 
    THURSDAY 
    FRIDAY
    SATURDAY 
)
```

## 数组和切片

数组和切片：`len`求长度，下标从0开始，`cap` 获取容量。

- **数组**是具有相同唯一类型的一组已编号且**长度固定**的数据项序列

切片不要求指定长度；**数组复制（=）是拷贝，切片复制是引用**（因为切片存储的是指针、长度、容量三个信息）：

- 类型 `[n]T` 是一个有 n 个类型为 T 的值的数组，如 `var a [2]int`表示有2个元素的int型数组。数组的长度是类型一部分；

- slice指向一个序列的值，包含长度信息，[]T是一个元素类型为T的slice，如 `s := []int {1,2,3,4}`；

  - slice可以重新切片，创建一个新的slice值指向相同的数组；`s[low:high]`包含从low到 high-1 的元素，low、high可以是变量；

  - slice 由函数 make 创建。这会分配一个全是零值的数组并且返回一个 slice 指向这个数组：

    `a := make([]int, 5, 10) // len(a)=5, capacity=10, capacity可以省略；`

  - slice 的零值是nil，通过append可以向slice添加元素； `var s [] int; s = append(s, 2)`

- for 循环的 range 格式可以对 slice 或者 map 进行迭代循环，`for i, v := range pow{}`，其中i是下标，v是值；不需要索引时，可以用_代替i，不需要值时只需i即可；

- **一个数组变量表示整个数组，它不是指向第一个元素的指针**（不像 C 语言的数组）。 当一个数组变量被赋值或者被传递的时候，实际上会**复制整个数组**。因此可以传递指针，但记住数组指针不是数组。

- 切片操作（指下标取值）并不复制切片指向的元素。它创建一个新的切片并复用原来切片的底层数组。这使得切片操作和数组索引一样高效。因此，通过一个新切片修改元素会影响到原始切片的对应元素。

- **小的内存引用导致保存所有的数据**。根据情形拷贝数据到新的切片中。

### append 函数

```go
slice2 := append(slice1, 23, 15)
```

以上对切片 `slice1` 进行 `append` 操作。该操作遵循以下原则：

1. `append` 函数对一个切片 `slice1` 进行追加操作，并返回另一个长度为 `len(slice1) + 追加个数` 的切片，原切片不被改动，两个切片所指向的底层数组可能是同一个也可能不是，取决于第二条：
2. `slice1` 是对其底层数组的一段引用，若 append 追加完之后没有突破 `slice1` 的容量，则实际上追加的数据改变了其底层数组对应的值，并且 `append` 函数返回对底层数组新的引用（切片）；若 `append` 追加的数据量突破了 `slice1` 的最大容量（底层数组长度固定，无法增加长度赋予新值），则 `Go` 会在内存中申请新的数组（数组内的值为追加操作之后的值），并返回对新数组的引用（切片）。

## 函数

函数的定义： 类型写在变量名后面，类型一致，可以写成`(x, y int)`，返回值写在参数定义后面

```go
func add(x int, y int) int { return x + y}
// 给返回值命名，返回两个int值，不需要显示 return变量
func split(sum int) (x, y int) {x = … y = … return}
```

### defer

> 在同一个函数内部调用多个 defer 语句，执行顺序是<font color='red'>**后进先出（LIFO）**</font>

将一个函数调用的执行推迟到包含该 `defer` 语句的函数执行完毕，无论是正常返回还是panic 异常退出，都会被执行。

- 通常用于资源释放和清理，保证一定执行；
- **参数值的即时求值**：当 `defer` 语句被执行时，传递给它的**函数参数会被立即求值**，并存储起来；

## 映射Map

```go
m := map[int]int{1: 1, 2: 12, 3: 13}
value, ok := m[1]
```

`map[key]`：如果key不存在，则value为key的类型的默认值，ok为false；



## 结构体

可以直接引用子结构体的成员变量？

Go中没有类，可以在结构体中定义方法，*方法接收者* 出现在`func`关键字和方法名之间的参数中：值类型

1. 接收者为指针时，一少拷贝开销，二可以修改传进去的接收者指向的值；

```go
type Dog struct {
    X int
}; 
func (v *Dog) Abs() int { return -v.X }
// 使用
(&Dog{1}).Abs()
```

### 结构体嵌套和匿名成员

**匿名成员**：定义不带名称的结构成员，只需指定类型，可以**直接访问匿名的结构体的成员**，同样可以访问匿名成员的内部方法；

- 匿名成员具有隐式的名字，因此**不能在一个结构体中定义两个相同类型的匿名成员**，否则会冲突；

```go
type Point struct {
    X, Y int
}
type Circle struct {
    Point
    Radius int
}
func (p *Point) addOne() {}

var c Circle
c.X = 1 // 直接访问 X
(&cc).addOne() // 或者 c.addOne()，直接使用Point的方法
```


## 接口

接口：是有一组方法定义的集合，接口类型的值可以存放实现这些方法的任何值

```go
type Abser interface {          
	Abs() int
}
var a Abser  // a可以赋值为任意实现Abs() float64方法的类型值
// 类型通过实现方法来实现接口，没有显示声明必要，因此无需关键字"implements"。
a = &Dog{2}  // a = Dog(2)是错的，因为没有定义Dog类型的Abs方法
// 接口类型断言： 
value, ok := element.(T) // 将element当作T类型，value是转换后结果，ok表示成功与否，nil表示成功；
```


## 模块（包）

> Golang1.11版本之前如果我们要自定义包的话必须把项目放在 GOPATH 目录。
>
> Go1.11版本之后无需手动配置环境变量，使用 go mod 管理项目，也不需要非得把项目放到 GOPATH

*GO111MODULE* 有三个值：off, on和auto（默认值）。

- *GO111MODULE=off*，无模块支持，go命令行将不会支持module功能，寻找依赖包的方式将会沿用旧版本那种通过vendor目录或者GOPATH模式来查找

- *GO111MODULE=on*，模块支持，go命令行会使用modules，而不会去GOPATH目录下查找。

- *GO111MODULE=auto*，默认值，go命令行将会根据当前目录来决定是否启用module功能。

  这种情况下可以分为两种情形：

  - 当前目录在GOPATH/src之外且该目录包含go.mod文件，开启模块支持。
  - 当前文件在包含go.mod文件的目录下面。

```shell
go mod init demo

# demo为项目文件夹,生成go.mod文件如下：
#######
# module demo
# go 1.17
#######
```



### init函数

- `init`函数用于包的初始化，如初始化包中的变量，这个初始化在`package xxx`的时候完成，也就是在`main`之前完成；
- 每个包可以拥有多个`init`函数， 每个包的源文件也可以拥有多个`init`函数；
- 同一个包中多个`init`函数的执行顺序是没有明确定义的，但是**不同包的`init`函数是根据包导入的依赖关系决定的；**
- `init`函数不能被其他函数调用，其实在`main`函数之前自动执行的。

示例：先执行init，再执行main

```go
package main 

import "fmt"

func main() {
    fmt.Println("do in main")
}


func init() {
    fmt.Println("do in init1")
}

func init() {
    fmt.Println("do in init2")
}  
```

### 引用

`import _ " "`

**不允许导入不使用的包**。但是有时候我们导入包只是为了做一些初始化的工作，这样就应该采用`import _ " "`的形式



### 可见性

在 Go 中，**首字母大写的名称是被导出的**，如`math.Pi`；（首字母大写代表对外部可见，首字母小写代表对外部不可见，适用于所有对象，包括方法和成员）

变量的可见性：

- 声明在**函数内部，是函数的本地值**，类似 private；
- 声明在**函数外部，是对当前包可见（包内所有.go文件都可见）的全局值**，类似protect；
- 声明**在函数外部且首字母大写是所有包可见的全局值**，类似public；

函数的可见性：

- 小写开头，则包内可见；
- <font color='red'>**大写开头，则全局可见**</font>；



## 错误和异常

`error`类型表示错误，值位`nil`表示没有错误；

`panic`表示异常，可以通过`recover`恢复；

golang提供了 recover 机制进行断点调试

```go
func A() { 
    defer func() { 
        if r := recover(); r != nil { 
            fmt.Println("Recovered in f", r) // 断点设于此，当发生error的时，GoLand可保存error发生时的现场
         } 
    } () 
    // 正常的函数代码逻辑
}
```

