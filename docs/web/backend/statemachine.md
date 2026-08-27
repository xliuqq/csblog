# 状态机

> 有限状态机（FSM）：Finite State Machine

`StateMachine<States, Events> `

- States：状态

- Events：事件

  定义 transitions ：

- 当前状态 + 事件 => 下一个状态

  定义动作：

- 进入状态的动作、离开状态的动作、执行事件的动作；

## [stateless4j](https://github.com/stateless4j/stateless4j)

> Lightweight Java State Machine.

- 无其他依赖；
- 足够轻量，创建StateMachine实例开销小；
- 支持目标状态的动态设置（即由函数确定；
- Action 同步且在调用线程中执行（即无线程池）；
- 持久化状态需应用端实现，实现比较麻烦；



## [Spring Statemachine](https://github.com/spring-projects/spring-statemachine)

> a common infrastructure to work with state machine concepts in Spring applications.

- 支持持久化；

- 分层状态机结构；

- 支持分布式状态机；

- 注解声明，功能丰富，同时上手较复杂；

- 状态机实例不能单例使用，线程不安全，配合工厂模式；

  每次请求**都要先构造出一个状态机实例**（因为有状态），并重置为之前的状态，然后再进行状态流转，如果构造状态机复杂，会影响系统的QPS，大量的状态机实例也可能会带来GC等问题。

- 使用工厂方式对于类似订单等场景StateMachineFactory缓存订单对应的状态机实例意义不大



## [squirrel](https://github.com/hekailiang/squirrel)

> a State Machine library, which provided a lightweight, easy use, type safe and programmable state machine implementation for Java.

- StateMachine实例创建开销小，设计上就不支持单例复用；
- 支持层次化状态机；
- 支持 外部状态转换（即状态发生变更）和内部状态转换（即发生事件但状态不变更）；
- 支持条件转换，既满足某个条件，才进行状态转换；
- Action 的执行，支持同步和异步；
- 支持注解配置 onEntry/OnExit 和 Transition；
- 对 Action 的执行，和Transition的执行，都定义相关的Before/After的Hook；



配置：

- 默认是一个线程的ExecutorServices，且 Action 默认为同步；