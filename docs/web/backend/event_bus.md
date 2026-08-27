# 事件处理

> Guava 中 EventBus ：事件处理机制，观察者模式（生产/消费者编程模型），处理一些异步任务，**应用于进程内通信**。

## Listener

> Receive and handle events.

通过`@Subscribe` 注解将**任意的类的方法**变为一个Listener

- 方法只能有一个参数表示 Event，不同类型的参数表明监听不同的类型的事件；
- `AllowConcurrentEvents`注解，标记方法是否线程安全；

```java
@Subscribe
@AllowConcurrentEvents
public void onEvent(ServiceRemoveEvent event) {
}
```

## EventBus

> 事件总线，线程安全。

register 方法注册 listener：*为什么通过反射，而不是通过接口？*

- 获取类的所有public 方法，**缓存所有**标记了  @Subscribe 注解的方法；
  - 将 method 的第一个入参，作为事件类型，区分不同的 listener；
- 根据 Listener 的**参数类型的不同**，分别向不同的Subscribe发送不同的消息。

```java
public static void main(String[] args) {
    final EventBus eventBus = new EventBus();    // 注册Listener，获取 @Subscribe 的 method 注解
    eventBus.register(new SimpleListener());
    System.out.println("post the simple event.");
    // 向订阅者发送消息
    eventBus.post(new ServiceRemoveEvent("Service removed Event"));
}
```

### Dispatcher

> Handler for dispatching events to subscribers, providing different event ordering guarantees that make sense for different situations.

EventBus 的 post 方法调用，会调用 `Subscriber` 的 `dispatchEvent` 方法；

不同线程 Post Event 事件的顺序保证：跟 Executor 是正交的关系（Executor 控制的是执行顺序，Dispatcher控制的是处理的顺序）

- `PerThreadQueuedDispatcher`：对已在调度事件的线程上重新发布的事件进行排队，以保证在单个线程上发布的所有事件都按发布顺序发送给所有订阅者。
  - `通过 ThreadLocal<Queue> + ThreadLocal<Boolean>` 实现，如果线程在处理事件时再发事件，则事件会按照广度优先的顺序处理；
  - bool 变量用于 `avoid reentrant event dispatching`，即递归事件处理时直接返回，再第一次事件处理通过 queue 遍历再处理；
- `ImmediateDispatcher`：立即给 Subscriber 执行，但没有可以直接使用的调用方法；
  - 如果线程在处理事件时再发事件，按照深度优先的顺序处理，
- `LegacyAsyncDispatcher`：concurrentHashMap 缓存事件，没有任何并发保证；
  - For async dispatch, an immediate dispatcher should generally be preferable

当使用 `PerThreadQueuedDispatcher`和`directExecutor`时（在发布事件的同一线程上进行调度），**yields a breadth-first dispatch order on each thread**

- 如果 事件A的订阅者 post 事件B和C，那么所有事件A的订阅者都会在任何事件B和C的订阅者之前被调用；
- all subscribers to a single event A will be called before any subscribers to any events B and C that are posted to the event bus by the subscribers to A.

### Subscriber

> A subscriber method on a specific object, plus the executor that should be used for dispatching events to it.

封装 `@Subscribe` 注解的方法，

- 方法中是否有 `@AllowConcurrentEvents` 注解，则为 `SynchronizedSubscriber` 类；

- 提供 `dispatchEvent` 方法，用于在 Executor 中执行被注解的方法；

### Executor

> 执行 @Subscribe 的线程。

默认为 `MoreExecutors.directExecutor()`：

- 枚举实现的单例，在调用方的线程中直接执行；

## AsyncEventBus

- 将 Dispatcher 的实例变成 LegacyAsyncDispatcher；
- 需要显式传入 Executor；


