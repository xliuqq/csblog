# Application Binary Interface

> [Linux Foundation Referenced Specifications](https://refspecs.linuxfoundation.org/)
>
> ABI 是编译器和链接器遵守的一组规则，以让编译后的程序可以正常工作

二进制兼容性：假设你的应用程序引用的一个库某天更新了，虽然 API 和调用方式基本没变，

- 需要重新编译你的应用程序才能使用这个库，那么一般说这个库是 Source compatible；
- **不需要重新编译应用程序就能使用新版本的库**，那么说这个库跟它之前的版本是二进制兼容。

ABI里包含很多方面的内容：

- 最大和最重要的部分是规定函数的调用顺序，也称为“调用约定”，标准化如何将“函数”转换为汇编代码。
- 规定了库中公开函数的name（如printf）如何表示，以便在链接后可以正确的调用这些库函数并接收参数。
- 规定可以使用什么类型的数据类型、它们必须如何对齐以及其他低级细节。
- 涉及操作系统的内容，如可执行文件的格式，虚拟地址空间布局，还有Program Loading and Dynamic Linking等细节

## Name mangling

C++ 引入（重载）的概念，其核心思想是把函数的名字、参数等信息（或者叫函数签名）编码成一个具有唯一性的字符串，用作链接符号；这样就能在编译期完成检查，从而避免运行时出问题（如`_ZN9wikipedia7article6formatEv`）。



## Calling Convention

函数调用：**同一个平台下同一个编译器的调用约定是一样的，不同平台可能不一样**。

- 调用者要把参数和返回点地址传递给函数，函数要把返回值传递给调用者；
  - 放栈和寄存器里了。哪些参数放栈顶，哪个参数先入栈，哪些参数放寄存器里，放哪个寄存器，返回地址放哪，返回值放哪，这都是调用约定的一部分
- 调用者在调用前后，某些寄存器里的值是还有用的，而函数内部也可能要用这个寄存器。
  - caller-saved 或者 callee-saved，也是调用约定的一部分



## 注意事项

- 开发二进制兼容的库的时候，一般使用 extern "C" 来抑制 Name mangling；
- 开发二进制兼容的库的时候，一定要避免使用 STL，因为不同的 C++ 编译器、不同版本的 C++ 编译器携带的 STL 不具备二进制兼容性，甚至同一个版本的 C++ 编译器用户也可能使用不同的 STL 替代自带的 STL
- 二进制兼容的接口应该只使用 int32、double 等基础数据类型，使用确定的 struct 甚至完全不使用 struct、只提供抽象的 handle，或者纯抽象接口
- 内存分配和释放不应该跨越 DLL Boundary，无法确定不同的模块内部使用的内存分配器是否相同；



## 实践

 针对 library （主要是 shared library，即动态链接库）的 ABI (application binary interface)，当 library 升级时，依赖该库的二进制不需要改动。

- non-virtual 函数比 virtual 函数更健壮：因为 **virtual function 是 bind-by-vtable-offset，而 non-virtual function 是 bind-by-name**。

### 二进制代码不兼容例子

- 给函数增加**默认参数**，现有的可执行文件无法传这个额外的参数；
- 增加**虚函数**，会造成 vtbl 里的排列变化（不要考虑“只在末尾增加”这种取巧行为，因为你的 class 可能已被继承。）；
- 增加**默认模板类型参数**，比方说 `Foo<T>` 改为 `Foo<T, Alloc=alloc<T>>`，这会改变 name mangling；

给 class Bar 增加数据成员，造成 sizeof(Bar) 变大，以及内部数据成员的 offset 变化：

- 不安全（不兼容）的客户端使用：

  - 客户代码里有 `new Bar`，那么肯定不安全；

  - 客户代码里有 `Bar* pBar; pBar->memberA = xx`，那么肯定不安全，memberA 的新 Bar 的偏移可能会变；

  - 如果客户调用 `pBar->setMemberA(xx);` 而 `Bar::setMemberA()` 是个 inline function，那么肯定不安全，因为偏移量已经被 inline 到客户的二进制里。

- 兼容的客户端使用：
  - 通过成员函数访问对象的数据成员，且该成员函数定义在 cpp 中，不是内联函数；

### 解决办法

第一种类似**桥接**设计模式：间接调用实现类实现，可能有一定的性能损失；

- 接口类：只定义 non-virtual 接口，只包含 `Impl *` 的私有成员；
- 接口类新增函数，只要新增 non-virtual 接口；

第二种：

- 对于接口类 Bar：

  - 所有的成员都是私有成员，并通过非内联的成员函数调用；
  - 修改时，只新增 non-virtual 接口；

  - **不提供公共的构造函数和析构函数**，而是提供工厂方法返回`Bar *`；

- 对于客户端要求：
  - 客户端不需要用到 `sizeof(Bar)`或者接口类不要添加新成员；
