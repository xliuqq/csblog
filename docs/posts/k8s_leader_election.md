---
date: 2024-01-03
readtime: 20
categories:
  - K8s
---



# K8s  中 leader election 选举原理

在开发CRD时，定义 `controller` 的时候，会看到如下代码

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    // 是否进行 leader 选举
    LeaderElection:          enableLeaderElection,
    // Namespace and name
    LeaderElectionNamespace: leaderElectionNamespace,
    LeaderElectionID:        "alluxio.data.fluid.io",
    // ...
})
```

对于有状态组件来说，实现高可用一般来说通过选主来达到同一时刻只能有一个组件在处理业务逻辑。

这里，会比较好奇这个选举是如何实现的，接下来的内容便从源码的角度进行解读。

<!-- more -->



## Controller Manager 中的选举

跟随`NewManager`的调用链，到

```go
// newResourceLock 方法默认是 leaderelection.NewResourceLock 方法
resourceLock, err := options.newResourceLock(leaderConfig, recorderProvider, leaderelection.Options{
    LeaderElection:             options.LeaderElection,
    LeaderElectionResourceLock: options.LeaderElectionResourceLock,
    LeaderElectionID:           options.LeaderElectionID,
    LeaderElectionNamespace:    options.LeaderElectionNamespace,
})
```

`leaderelection.NewResourceLock`方法中，最终调用 client go 库中的 `resourcelock.New`方法：

- `LeaderElectionResourceLock`默认为 `ConfigMapsLeasesResourceLock`；
- `id`是 `hostname_$uuid`构成；

```go
return resourcelock.New(options.LeaderElectionResourceLock,
		options.LeaderElectionNamespace,
		options.LeaderElectionID,
		corev1Client,
		coordinationClient,
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: recorderProvider.GetEventRecorderFor(id),
		})
```



## Client Go 中的 Leader Election

> [官方示例](https://github.com/kubernetes/client-go/blob/master/examples/leader-election/main.go)
>
> - 官方支持锁的类型有 `LeaseLock`, `configmapLock`,`endpointLock`和`MultiLock`；
> - `MultiLock`是用来将原先的 `configmapLock`和`endpointLock`迁移过来而设计；
>   - `configmapLock`->`multiLock(ConfigMapsLeasesResourceLock)`->`LeaseLock`
> - 以下分析，基于 client-go@v0.24.17 版本

构建锁实例：

- `LeaseLock` 可以直接构造，而其它的锁要通过`resourcelock.New`工厂方法构造；
- **一般使用 `LeaseLock`**，因为这种类型更不常见（用户不会使用它），并且所观察的对象会更少；
- **K8s的对象（如`LeaseLock`）是不会被删除**，即使所有的进程都退出，下次启动后仍可以正常选取；

```go
lock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      leaseLockName,
        Namespace: leaseLockNamespace,
    },
    Client: client.CoordinationV1(),
    LockConfig: resourcelock.ResourceLockConfig{
        Identity: leaderId,
    },
}
```

通过 `leaderelection.RunOrDie`进行leader election，并执行业务逻辑

```go
// start the leader election code loop
leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    // 上文构建的 lock
    Lock: lock,
    // IMPORTANT: you MUST ensure that any code you have that
    // is protected by the lease must terminate **before**
    // you call cancel. Otherwise, you could have a background
    // loop still running and another process could
    // get elected before your background loop finished, violating
    // the stated goal of the lease.
    ReleaseOnCancel: true,
    // 锁的持续时间（TTL）
    LeaseDuration:   60 * time.Second,
    // 锁的续约时执行的超时时间
    RenewDeadline:   15 * time.Second,
    // 申请锁/续约锁（acquire）的循环周期
    RetryPeriod:     5 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            // we're notified when we start - this is where you would
            // 执行业务代码，会被单独的协程调用
            run(ctx)
        },
        OnStoppedLeading: func() {
            // we can do cleanup here
            klog.Infof("leader lost: %s", leaderId)
            os.Exit(0)
        },
        OnNewLeader: func(identity string) {
            // we're notified when new leader elected
            if identity == leaderId {
                // I just got the lock
                return
            }
            klog.Infof("new leader elected: %s", identity)
        },
    },
})
```

`RunOrDie`的核心逻辑在`Run`方法调用中

```go
func (le *LeaderElector) Run(ctx context.Context) {
    // 从 panic 中 recover 并打印堆栈
	defer runtime.HandleCrash()
    // 调用 OnStoppedLeading 的 callback
	defer func() {
		le.config.Callbacks.OnStoppedLeading()
	}()
	// 申请锁，如果申请不到则会阻塞
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel() 
	// 另起协程，调用 OnStartedLeading 的 callback
	go le.config.Callbacks.OnStartedLeading(ctx)
	// 续约锁，一直阻塞，如果续约失败（则进程应该直接退出，即 Die）
	le.renew(ctx)
}
```

### 申请锁

源代码如下

```go
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	desc := le.config.Lock.Describe()
	klog.Infof("attempting to acquire leader lease %v...", desc)
    // 循环执行函数一直到 ctx.Done() 这个 channel closed，循环周期为 le.config.RetryPeriod
	wait.JitterUntil(func() {
        // 核心代码，申请锁或者续约锁，在一个函数中
		succeeded = le.tryAcquireOrRenew(ctx)
        // 如果 leader 发生变化，则调用 onNewLeader 的 callback
		le.maybeReportTransition()
        // 没有申请到锁则重试
		if !succeeded {
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
        // 如果申请到锁，调用 cancle()，会退出 wait 循环
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}
```

核心函数`tryAcquireOrRenew`源码如下：

- 申请锁或者续约锁，

```go
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
        // 自己为 leader
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1. 获取锁
	oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
	if err != nil {
        // 锁存在，但存在其他错误，执行失败
		if !errors.IsNotFound(err) {
			klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}
        // 锁不存在，创建锁（k8s api 保证只有一个能执行成功）
		if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
			klog.Errorf("error initially creating leader election record: %v", err)
			return false
		}
		// 更新内存的当前锁持有信息
		le.setObservedRecord(&leaderElectionRecord)
		return true
	}
	// 2. 更新自己的 observedRecord（包括锁的当前持有者） 和 observedTime
	if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
		le.setObservedRecord(oldLeaderElectionRecord)
		le.observedRawRecord = oldLeaderElectionRawRecord
	}
    
    // 3. 虽然获取到锁（仅获取到对象）不代表能够持有该锁执行业务逻辑，此时需要判断锁是否由自己持有
    
    // 3.1 所得持有者不是自己且锁还没过期，则获取失败
	if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) && !le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	if le.IsLeader() {
    	// 3.2 锁的持有者是自己（无论锁是否过期），更新锁的获取时间
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
        // 3.3 锁的持有者不是自己且锁过期了，锁的所有权转换次数+1
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// 更新锁， 即使存在并发竞争（当锁过期时）：
    // 1）锁的持有者更新，3.2 逻辑；
    // 2）锁的非持有者申请，3.3 逻辑；
    // k8s 的 API 保证仅有一个客户端能够执行成功
	if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		return false
	}

	// 更新内存的当前锁持有信息
	le.setObservedRecord(&leaderElectionRecord)
	return true
}
```

### 续约锁

```go
// 此处的 ctx 是 OnStartedLeading callback 函数的参数的 ctx
// 如果 callback 中 ctx 调用 cancel 会执行所有 子 context 的cancel
func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx
	defer cancel()
    // 每个 RetryPeriod 续约锁，直到 ctx 被 cancel()
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
        // 续约锁直到1）返回true，即续约成功 或2）锁超时 或3）返回 err（此处一直为nil，不会发生）
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			return le.tryAcquireOrRenew(timeoutCtx), nil
		}, timeoutCtx.Done())

        // 续约的时候没抢到锁，汇报锁的状态转移，调用 onNewLeader 的 callback
		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
        // 续约锁成功，则等待下次循环
		if err == nil {
			klog.V(5).Infof("successfully renewed lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("stopped leading")
		le.metrics.leaderOff(le.config.Name)
		klog.Infof("failed to renew lease %v: %v", desc, err)
        // 续约锁失败，退出 wait，进程应该直接 Die
		cancel()
	}, le.config.RetryPeriod, ctx.Done())

	// 如果此时我们还持有锁，但是续约失败，则释放该锁
	if le.config.ReleaseOnCancel {
        // 函数内将 LeaseDurationSeconds 置为 1s（即很快过期），然后 Update Lock
        // 即使更新失败，也没有影响，等待锁过期即可
		le.release()
	}
}
```



## 锁的实现

`resourcelock.Interface` 接口定义了锁的相关接口

```go
// a common interface for locking on arbitrary resources used in leader election
type Interface interface {
	// Get returns the LeaderElectionRecord
	Get(ctx context.Context) (*LeaderElectionRecord, []byte, error)

	// Create attempts to create a LeaderElectionRecord
	Create(ctx context.Context, ler LeaderElectionRecord) error

	// Update will update and existing LeaderElectionRecord
	Update(ctx context.Context, ler LeaderElectionRecord) error

	// RecordEvent is used to record events
	RecordEvent(string)

	// Identity will return the locks Identity
	Identity() string

	// Describe is used to convert details on current resource lock
	// into a string
	Describe() string
}
```

 `LeaseLock`结构体对该接口进行了实现，主要的`Get`, `Create`和 `Update`方法，就是对`Lease` Resource 的 `Get`, `Create`和 `Update`的操作。

### 