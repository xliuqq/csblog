# 状态模式

## 意图

允许对象**在内部状态发生改变时改变它的行为**，对象看起来好像修改了它的类。

**解决：**对象的行为依赖于它的状态（属性），并且可以根据它的状态改变而改变它的相关行为。



## 适用性

1、行为随状态改变而改变的场景。 

2、条件、分支语句的代替者。

通常在以下情况下可以考虑使用状态模式。

- 当一个对象的行为取决于它的状态，并且它必须在运行时根据状态改变它的行为时，就可以考虑使用状态模式。
- 一个操作中含有庞大的分支结构，并且这些分支决定于对象的状态时

## 类

![状态模式的结构图](.pics/state/state.png)

1. **环境类**（Context）角色：也称为上下文，它定义了客户端需要的接口，内部维护一个当前状态，并负责具体状态的切换。
2. **抽象状态**（State）角色：定义一个接口，用以封装环境对象中的特定状态所对应的行为，可以有一个或多个行为。
3. **具体状态**（Concrete State）角色：实现抽象状态所对应的行为，并且在需要的情况下进行状态切换。



## 优劣势

**优点：** 

- 结构清晰，状态模式将与特定状态相关的行为局部化到一个状态中，并且将不同状态的行为分割开来，满足“单一职责原则”。
- 将状态转换显示化，减少对象间的相互依赖。将不同的状态引入独立的对象中会使得状态转换变得更加明确，且减少对象间的相互依赖。
- 状态类职责明确，有利于程序的扩展。通过定义新的子类很容易地增加新的状态和转换。
- 可以让多个环境对象共享一个状态对象，从而减少系统中对象的个数。

**缺点：** 

- 状态模式的使用必然会增加系统类和对象的个数；
- 状态模式的**结构与实现都较为复杂**，如果使用不当将导致程序结构和代码的混乱；
- 状态模式**对"开闭原则"的支持并不太好**，对于可以切换状态的状态模式，增加新的状态类需要修改那些负责状态转换的源代码，否则无法切换到新增状态，而且修改某个状态类的行为也需修改对应类的源代码。