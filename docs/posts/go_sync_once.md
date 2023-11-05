---
date: 2023-10-23
categories:
  - 并发
---

# 源码分析之Go Once

> Go 中的 Atomic Values 等价于 C++ 的顺序一致性 atomics，等价于 Java 中的 `volatile`变量；

在看 Go 中 `sync.once`包中的源码实现时，疑问为什么要用`atomic`的 load 和 store，而不能直接读取和赋值。

```go
if o.done == 0 {
    // 为什么不使用? defer func(){o.done == 1}()
    defer atomic.StoreUint32(&o.done, 1)
    f()
}
```



<!-- more -->



## Once 代码

> Go 1.18 中的源码

```go linenums="1"
type Once struct {
	done uint32
	m    Mutex
}
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}
func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
        // defer func(){o.done == 1}()
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

Once 的`Do`保证只会执行函数一次，且**其它go routine 会等待直到该函数执行完**。

- **第15行**使用原子性 Store，而不是第14行的形式，是防止出现**指令重排序**的情况，导致先执行 `o.done == 1`，再执行`f()`
- 类比于 Java 实现的双重检查锁（DCL），`done`这里使用 atomic 等价于 `volatile`语义；

代码的官方解释见：https://codereview.appspot.com/4641066/#msg14

Go 的内存模型：https://go.dev/ref/mem
