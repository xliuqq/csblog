# Channel

> channel 的零值为 nil：
>
> - 在 nil 通道上发送和接收将会永远阻塞；
> - select语句中，如果通道为nil，将永远不会被选择；

 Go 语言提供了一种不同的**并发模型**，即通信顺序进程（**Communicating sequential processes**，**CSP**）。

Goroutine 和 Channel 分别对应 CSP 中的实体和传递信息的媒介，Goroutine 之间会通过 Channel 传递数据。

## 基本使用

目前的 Channel 收发操作均遵循了**先进先出**的设计，具体规则如下：

- 先从 Channel 读取数据的 Goroutine 会先接收到数据；
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

用法：

```go
// 创建 channel，第二个参数代表Channel的缓存的大小，不设置或者为0，
// 表示没有缓存，只有sender和receiver都准备好了后它们的通讯(communication)才会发生(Blocking)
ch := make(chan int)
// 从Channel ch中接收数据，并将数据赋值给v
x := <- ch
// 发送值 x 到 Channel ch 中
ch <- x
//
close(ch)
```

- 如果channel c 已经被关闭,继续往它发送数据会导致`panic: send on closed channel`;
- 但是从这个关闭的channel中不但可以读取出已发送的数据，还可以**不断的读取零值**；
- 如果通过`range`读取，**channel关闭后for循环会跳出**；
- `i, ok := <-c`可以查看 Channel的状态（`i`为 0， `ok`为 false）；

### 同步

channel 用在 **goroutine 之间的同步**。
下面的例子中main goroutine通过done channel等待worker完成任务。 worker做完任务后只需往channel发送一个数据就可以通知main goroutine任务完成。

```go
import (
    "fmt"
    "time"
)
func worker(done chan bool) {
    time.Sleep(time.Second)
    // 通知任务已完成
    done <- true
}
func main() {
    done := make(chan bool, 1)
    go worker(done)
    // 等待任务完成
    <-done
}
```

## select

`select` 也能够让 Goroutine 同时等待多个 Channel 可读或者可写，**在多个文件或者 Channel状态改变之前，`select` 会一直阻塞当前线程或者 Goroutine**。

```go
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {  // 循环计算
		select {
		case c <- x:  // 计算的值发送到 channel c 中
			x, y = y, x+y
		case <-quit:  // 如果 channel quit 有输入，则退出
			fmt.Println("quit")
			return
		}
	}
}
```

上述控制结构会等待 `c <- x` 或者 `<-quit` 两个表达式中任意一个返回。

无论哪一个表达式返回都会立刻执行 `case` 中的代码，当 `select` 中的两个 `case` 同时被触发时，会**随机**执行其中的一个。

### 超时

select 如果没有case需要处理，select语句就会一直阻塞着。这时候我们可能就需要一个超时操作，用来**处理超时**的情况。

- `time.After`方法，返回一个类型为`<-chan Time`的单向的channel，在指定的时间发送一个当前时间给返回的channel中。

```go
import "time"
import "fmt"
func main() {
    c1 := make(chan string, 1)
    go func() {
        time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    select {
    case res := <-c1:
        fmt.Println(res)
    case <-time.After(time.Second * 1):
        fmt.Println("timeout 1")
    }
}
```



## Timer和Ticker

`timer`是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间，它提供一个Channel，在将来的那个时间那个Channel提供了一个时间值。

- 单纯的等待的话，可以使用`time.Sleep`来实现；

```go
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
    fmt.Println("Timer 2 expired")
}()
stop2 := timer2.Stop()
if stop2 {
    fmt.Println("Timer 2 stopped")
}
```

`ticker`是一个定时触发的计时器，它会以一个间隔(interval)往Channel发送一个事件(当前时间)，而Channel的接收者可以以固定的时间间隔从Channel中读取事件。下面的例子中ticker每500毫秒触发一次，你可以观察输出的时间。

- 类似timer, ticker也可以通过`Stop`方法来停止。一旦它停止，接收者不再会从channel中接收数据了。

```go
ticker := time.NewTicker(time.Millisecond * 500)
go func() {
    for t := range ticker.C {
        fmt.Println("Tick at", t)
    }
}()
time.Sleep(time.Second * 2)
ticker.Stop()
```

