# 代理模式

## 意图

为其它对象提供一种代理以控制对这个对象的访问。



## 适用性

- 远程代理：代理远程的对象，屏蔽网络等细节，比如RPC；
- 虚代理：需要创建开销很大的对象，让调用者先持有一个代理对象，但真正的对象尚未创建；
- 保护代理：控制对原始对象的访问，用于权限控制等；
- 智能指引：访问对象时执行附加操作，比如引用计数等；
- Copy-on-write代理：减少不必要的拷贝，只有在修改时才真正拷贝；

## 类图

![代理模式](.pics/proxy/proxy.png)

## 优缺点

**优点：** 1、职责清晰。 2、高扩展性。 3、智能化。

**缺点：** 1、由于在客户端和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢。 2、实现代理模式需要额外的工作，有些代理模式的实现非常复杂。