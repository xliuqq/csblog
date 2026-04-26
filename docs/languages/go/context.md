# Context

上下文 `context.Context` Go 语言中用来**设置截止日期、同步信号，传递请求相关值**的结构体。

可能会创建多个 Goroutine 来处理一次请求，而 [`context.Context`](https://draveness.me/golang/tree/context.Context) 的作用是在**不同 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期**。

<img src="pics/golang-context-usage.png" alt="golang-context-usage" style="zoom:67%;" />

每一个 `context.Context` 都会从最顶层的 Goroutine 一层一层传递到最下层。`context.Context` 可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层。



## 构造

> 不可变性

### context.BackGround

- returns a non-nil, empty Context
- never canceled, has no values, and has no deadline
- as the top-level Context for incoming requests

### context.TODO

- returns a non-nil, empty Context
- never canceled, has no values, and has no deadline
- use context.TODO when it's unclear



### context.WithValue

> a **copy** of parent in which the value associated with key is val.
>
> - Users of WithValue should define their own types for keys, not use built-in type or string ;

```go
// ctx 中存储了 "a" 为 1
ctx := context.withValue(context.Background(), "a", 1)
```

实际获取的时候，是从当前层开始逐层的进行向上递归，直至找到某个匹配的key 或者返回 nil。



### context.WithCancel

> a copy of parent with a **new Done channel**.
>
> The returned context's **Done channel is closed when the returned cancel function is called or when the parent context's Done channel is closed**, whichever happens first.

取消某个parent的goroutine, 实际上会**递归层层cancel掉自己的child context的done chan**从而**让整个调用链中所有监听cancel的goroutine退出**

- 当代码执行完成后，应当立即执行 `cancel` 函数（这里的cancel 有 complete的含义）；

```go
type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields 保护属性
    done     chan struct{}         // created lazily, closed by first cancel call
    // 记录子 context 的cancel
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}
```



### context.WithTimeout

实现原理：**启动一个定时器，然后在超时的时候，直接将当前的context给cancel掉**，就可以实现监听在当前和下层的`context.Done()`的`goroutine`的退出

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // timer定时器

    deadline time.Time //终止时间
}
```

示例：

- 即时超时返回了，但是子协程仍在继续运行，直到自己退出

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    // cancel 一定要执行
	defer cancel()

	go handle(ctx, 500*time.Millisecond)
	select {
	case <-ctx.Done():
		fmt.Println("main", ctx.Err())
	}
}

func handle(ctx context.Context, duration time.Duration) {
	select {
	case <-ctx.Done():
		fmt.Println("handle", ctx.Err())
	case <-time.After(duration):
        // 500ms < 1s, run here
		fmt.Println("process request with", duration)
	}
}
```

