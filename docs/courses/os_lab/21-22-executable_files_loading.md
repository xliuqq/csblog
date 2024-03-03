# 可执行文件与加载

> **背景回顾**：我们已经见识过在系统调用 API 和操作系统系统对象上层层封装得到的世界了。是时候实现一些 “真正” 的程序了——让我们看一看到底什么是**可执行文件**，以及它们是**如何被操作系统加载**的。



## 可执行文件



### 什么是可执行文件

> ELF 格式（Linux）
>
> - `readelf` 

可执行文件：

- 一个操作系统中的对象（文件）
- 一个字节序列（可以当文本编辑）
- 描述状态机初始状态的**数据结构**



### 作为”数据结构“的可执行文件

> RTFM: [System V ABI](https://jyywiki.cn/pages/OS/manuals/sysv-abi.pdf)
>
> - [binutils](https://www.gnu.org/software/binutils/) 中的工具可以让我们查看其中的重要信息

状态机初始状态的描述：

- 内存中各段的位置和权限
- 初始的 PC 在 ELF Header 的 entry
- 寄存器和栈由操作系统决定（见上一节的 env.c）

状态机的描述：

- 代码



### 加载最小可执行文件

> 实验代码：通过 ChatGPT4 进行代码生成

最小可执行文件

- 代码在内存中
- PC 指向第一条指令
- 除此之外，任何初始状态都行

直接把代码 `mmap` 到内存

- 然后跳转过去即可
- (你可以想象 execve 里就做了这件事)



## 静态链接和加载

### 在操作系统上实现 ELF Loader

加载器 (loader) 的职责

- 解析数据结构
- 创建进程初始状态
  - argv, envp, ...
  - 再一次，[System V ABI](https://jyywiki.cn/pages/OS/manuals/sysv-abi.pdf)
- 跳转执行

代码示例

- 能正确处理参数/环境变量 [env.c](https://jyywiki.cn/pages/OS/2023/v/env.c)



### Boot Block Loader

加载操作系统内核？

- 也是一个 ELF 文件
- 解析数据结构 + 复制到内存 + 跳转

之前给大家看过一眼

- 做得事情与静态加载器完全一样
- 没有 mmap 怎么办？
  - 用 I/O 指令把数据从磁盘搬到内存

### ELF 文件是如何生成的？

一个字节一个字节 “写出来” 的

- `printf("\x7f");`

**绝大部分指令都是编译时就完全确定**

- 但有少部分不能，例如 printf 的地址 (引用第三方)
  - call 会编译成 `e8 00 00 00`
    - 复习题：`offset 0` 是跳转到哪条指令，为什么？
  - 连接时需要 relocate
    - 这也是 ELF “数据结构” 的一部分
    - 你可以理解成 “`ADDR_OF(puts) + 4`” 这种运算规则



### 为什么 ICS 课的 “链接和加载” 如此僵硬？

 ELF 不是一个好的 “描述状态机数据结构” 的格式，彻底**违背了信息的局部性原则**：

- 含义隐晦的 `R_X86_64_32`, `R_X86_64_PLT32`
- 不靠解析根本不可能看懂
- 大量的 “指针” (人类无法阅读的偏移量)
- 等于直接让你去读一个内存数据结构的 core dump



## 动态链接和加载

### ”拆解应用程序“的需求

实现**运行库和应用代码分离**

- Linux 中绝大部分应用都是动态链接的
  - 系统中只有一份 libc （节省磁盘）
- Library 保持接口的向后兼容
  - **补丁发布**后不再需要**重编译所有依赖**的应用
  - [Semantic Versioning](https://semver.org/)：版本号规范（Major.Minor.Fix）
    - “Compatible” 是个有些微妙的定义

大型项目内部也可以内部分解

- 编译一部分，不用重新链接
- libjvm.so, libart.so, ...
  - NEMU: “把 CPU 插上主板”

### 动态链接：今天不讲 ELF

换一种方法

- 如果编译器、链接器、加载器都受你控制
- 你怎么设计、实现一个 “最直观” 的动态链接格式？
  - 再去考虑怎么改进它，你就得到了 ELF！
- **<font color=red>假设编译器可以为你生成位置无关代码 (PIC)</font>**

### 设计一个新的二进制文件格式

>动态链接的符号查表就行了嘛。

```ini
DL_HEAD

LOAD("libc.dl") # 加载动态库
IMPORT(putchar) # 加载外部符号
EXPORT(hello)   # 为动态库导出符号

DL_CODE

hello:
  ...
  call DSYM(putchar) # 动态链接符号
  ...

DL_END
```

存储保护和加载位置

- 允许将 .dl 中的一部分以某个指定的权限映射到内存的某个位置 (program header table)

允许自由指定加载器 (而不是 dlbox)

- 加入 INTERP

空间浪费

- 字符串存储在常量池，统一通过 “指针” 访问
  - 这是带来 ELF 文件难读的最根本原因

其他：不那么重要

- 按需 RTFM/RTFSC



```c
#define DSYM(sym)   *sym(%rip)
```

DSYM 是间接内存访问

```c
extern void foo();
foo();
```

一种写法，两种情况

- 来自其他编译单元 (静态链接)
  - 直接 PC 相对跳转即可
- 动态链接库
  - 必须查表 (编译时不能决定)



## 动态链接和加载原理

### 实现库的运行时加载

若干要素

- 编译成位置无关代码
  - `lea (0x400000), addr_of_x` (no)
  - `lea 12(%rip), addr_of_x` (yes)
  - 这是编译器可以实现的
- 对外部库函数的调用是查表的
  - `call TABLE(x)`
- 在运行 (加载) 时填表
  - 加载时把导出的符号填入 TABLE

我们 “发明” 了 **GOT (Global Offset Table)**

- 就是 TABLE

### 一个有趣的问题

```c
extern void foo();
```

编译器遇到函数调用，应该翻译成哪种指令？

- 如果 foo 来自同一个动态链接库，不用查表
  - `call foo`
- 如果 foo 来自另一个动态链接库
  - `call TABLE(foo)`

我们发明了 **PLT (Procedure Linkage Table)**：生成同样的 call 指令

- 编译器总是生成一个直接的 call
  - 来自另一个动态链接库：**call putchar@PLT**
  - 链接的时候增加间接跳转的只读代码

- 函数实在太多了
  - 每个都标记区分，太难看了

## ELF 动态链接与加载

> 更好的”知识网络“，`ldd`等相关命令 -> ”状态机初始状态“ 可视化 



27min47s
