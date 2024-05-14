# 可执行文件与加载

**本讲内容**：

- 可执行文件
- 静态/动态链接和加载
- 动态链接与加载
- ELF 的动态链接
- `LD_PRELOAD` Hooking

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

最小可执行文件（汇编实现的`HelloWorld`，`.text`段很简单）

- 代码在内存中
- PC 指向第一条指令
- 除此之外，任何初始状态都行

直接把代码 `mmap` 到内存

- 然后跳转过去即可 (你可以想象 execve 里就做了这件事)



## 静态链接和加载

### 在操作系统上实现 ELF Loader

加载器 (loader) 的职责（静态链接，所有的内容都在二进制文件里）

- 解析数据结构
- 创建进程初始状态
  - argv, envp, ...
  - 再一次，[System V ABI](https://jyywiki.cn/pages/OS/manuals/sysv-abi.pdf)
- **跳转执行**

代码示例：打印 environ 的二进制代码

- 能正确处理参数/环境变量 env.c

### Boot Block Loader

加载操作系统内核？

- 也是一个 ELF 文件
- 解析数据结构 + 复制到内存 + 跳转

之前给大家看过一眼

- 做得事情与静态加载器完全一样
- 没有 mmap 怎么办？
  - 用 I/O 指令把数据从磁盘搬到内存，读成自己序列，然后指针赋值（如何可执行？`mprotect`）；

### ELF 文件是如何生成的？

一个字节一个字节 “写出来” 的

- `printf("\x7f");`

**绝大部分指令都是编译时就完全确定**

```c
int bar();

int foo() {

        return bar() + 1;
}
// gcc -c test.c
// objdump -d test.o
Disassembly of section .text:

0000000000000000 <foo>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   b8 00 00 00 00          mov    $0x0,%eax
   9:   e8 00 00 00 00          callq  e <foo+0xe>
   e:   83 c0 01                add    $0x1,%eax
  11:   5d                      pop    %rbp
  12:   c3                      retq   
      
// readelf -a test.o
Relocation section '.rela.text' at offset 0x1c0 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000a  000900000004 R_X86_64_PLT32    0000000000000000 bar - 4
```

- 但有少部分不能，例如 printf 的地址 (引用第三方)
  - call 会编译成 `e8 00 00 00`
    - 复习题：`offset 0` 是跳转到哪条指令，为什么？
    - 下一条指令的开始（即本条指令的末尾），而不是本条指令的开始。（指令可以加前缀）
  - 连接时需要 relocate
    - 这也是 ELF “数据结构” 的一部分
    - 你可以理解成 “`ADDR_OF(puts) + 4`” 这种运算规则



### 为什么 ICS 课的 “链接和加载” 如此僵硬？

 ELF 不是一个好的 “描述状态机数据结构” 的格式，彻底**违背了信息的局部性原则**：

- 含义隐晦的 `R_X86_64_32`, `R_X86_64_PLT32`
- 不靠解析根本不可能看懂
- 大量的 “指针” (人类无法阅读的偏移量)
- 等于直接让你去读一个内存数据结构的 core dump

一个机会重新设计？

- 可以做个 JSON-based，可读的格式
  - FLE: A Fast, Legible, and Expedient Binary Format



## 动态链接和加载

### ”拆解应用程序“的需求

实现<font color='red'>**运行库和应用代码分离**</font>

- Linux 中绝大部分应用都是动态链接的：系统中只有一份 libc （节省磁盘）
- Library 保持接口的向后兼容：**补丁发布**后不再需要**重编译所有依赖**的应用
  - [Semantic Versioning](https://semver.org/)：版本号规范（Major.Minor.Fix）：“Compatible” 是个有些微妙的定义

大型项目内部也可以内部分解：编译一部分，<font color='red'>**不用重新链接**</font>

- `libjvm.so, libart.so, ...`
  - NEMU: “把 CPU 插上主板”

### 动态链接：今天不讲 ELF

> 每句话都没说错，但没人能跟上
>
> - 根本原因：概念上紧密相关的东西在实现中被 “拆散” 了

换一种方法

- 如果编译器、链接器、加载器都受你控制
- 你怎么设计、实现一个 “最直观” 的动态链接格式？
  - 再去考虑怎么改进它，你就得到了 ELF！
- **<font color=red>假设编译器可以为你生成位置无关代码 (PIC)</font>**

### 设计一个新的二进制文件格式（dl）

><font color='red'>**动态链接的符号查表**</font>就行了嘛。

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

编译器：`GCC, GNU as`

`binutils`

- `ld = objcopy` (偷来的)

- `as = GNU as` (偷来的)

- 剩下的就需要自己动手了

  - `readdl (readelf)`
  - `objdump`
  - 你同样可以山寨 `addr2line, nm, objcopy, ...`

  - 和最重要的<font color='red'>**加载器**</font>

### 解决 dl 文件的缺陷

#### 功能缺陷

存储保护和加载位置

- 允许将 `.dl` 中的一部分以<font color='red'>**某个指定的权限映射**</font>到内存的某个位置 (program header table)

允许自由指定加载器 (而不是 dlbox)

- 加入 `INTERP`

空间浪费

- 字符串存储在常量池，统一通过 “指针” 访问：这是带来 ELF 文件难读的最根本原因

其他：不那么重要

- 按需 RTFM/RTFSC

```c
#define DSYM(sym)   *sym(%rip)
```

#### 性能缺陷

DSYM 是间接内存访问

```c
extern void foo();
foo();
```

一种写法，两种情况

- 来自其他编译单元 (静态链接)：直接 PC 相对跳转即可
- 动态链接库：必须查表 (编译时不能决定)



## 动态链接和加载原理

### 实现库的运行时加载

若干要素

- 编译成位置无关代码：这是编译器可以实现的
  - `lea (0x400000), addr_of_x` (no)
  - `lea 12(%rip), addr_of_x` (yes)
- 对外部库函数的调用是查表的
  - `call TABLE(x)`
- 在运行 (加载) 时填表
  - 加载时把导出的符号填入 TABLE

我们 “发明” 了 **GOT (Global Offset Table)**：就是 TABLE

### 一个有趣的问题

```c
extern void foo();
```

编译器遇到函数调用，应该翻译成哪种指令？

- 如果 foo 来自同一个动态链接库，不用查表：`call foo`
- 如果 foo 来自另一个动态链接库：`call TABLE(foo)`

我们发明了 **PLT (Procedure Linkage Table)**：生成同样的 call 指令

- 编译器总是生成一个直接的 call
  - 来自另一个动态链接库：**call putchar@PLT**
  - <font color='red'>**链接的时候增加间接跳转的只读代码**</font>

- 函数实在太多了
  - 每个都标记区分，太难看了

## ELF 动态链接与加载

> 更好的”知识网络“，`ldd`等相关命令 -> ”状态机初始状态“ 可视化 

### Executable Linkable Format

ELF 和 “dl” 没有本质区别

- 当然，有海量工程实践上的细节
- 与 99.9% 的程序员无关

“状态机初始状态” → 能够可视化

- ldd - Print shared object dependencies

  - SEE ALSO 一直是手册里的宝藏

  - 指向了 ld.so (8)

对于一个动态链接的二进制文件，execve 后的第一条指令在哪里？

- `gdb`中 `starti`，`ld.so`中的`_start`；

这种问题，再也不需要老师教（从此 “知识” 不再是壁垒和禁区）

- `What are the first a few steps executed after execve() of a ELF dynamic link binary?`
- `How can I compile an ELF binary that use an alternative dynamic loader than the default ld.so?`
  - `-Wl,--dynamic-linker=`

### 重新思考 PLT 的设计

> 极致性能：云上linux虚拟机，单个应用直接在内核态执行。

```assembly
puts@PLT:
  endbr64
  bnd jmpq *GOT[n]  // *offset(%rip)
```

一个有趣 (且根本) 的问题

- **库函数调用看起来 “很浪费”**：连续的跳转
- 为什么不在加载时执行静态链接？
  - 把指令中的立即数替换成跳转地址，这样就避免了查表？
  - <font color='red'>**初始化开销 VS 查表的开销**</font>
  - 动态链接：复用节省空间，无论是磁盘空间和内存空间（共享动态库的代码段），减少物理页换入换出、增加缓存命中率
    - 内核中会为每个文件节点维持一个radix树（Page Cache），记录了为加载该文件已经分配的物理内存页框

### 代码解决了，数据呢？

如果我自己的共享库要使用数据？

```c
extern FILE *stdout;
extern char *__lib_private;
```

- 对于 stdout，无论多少库，都只有一个副本，必须查表
  - `stdout`这个变量是放在 libc.so 中，还是a.out？

- 对于 buf，可以直接翻译成 PC 相对寻址

这个问题 GPT-4 答错了

- `-fPIC` 默认会为**所有 extern 数据**增加一层间接访问
  - 此时共享库中只有它的地址，需要重定位
- `__attribute__((visibility("hidden")))`控制符号（如函数或变量）的链接可见性
  - 不会在共享库的符号表中导出，其他链接到该库的代码不能直接访问该符号

## LD_PRELOAD

### 链接 “修改” 过的 libc

- 不用像修改器那样 “入侵” 地址空间了
- 程序会主动把控制流交给我们

`LD_PRELOAD`: 在加载之前 preload

- 拦截程序对库函数的调用，如调试和内存检测、性能分析、兼容性修复等；

 `How can I hook calls to malloc and free using LD_PRELOAD?`

- 利用动态链接特性：符号先到先占坑
  - 先加载一个自己的库，占据符号
- 通过`dlsym`函数来获取原始`malloc`和`free`的地址

### 其他操作系统上的 Hooking

> Open Question: 如何反游戏外挂？

`Windows DLL Injection`：

- `DLL: Dynamic Link Library`

- 理论效果与 `LD_PRELOAD` 类似

Android

- LSPosed (Xposed 后继项目)
- Android App 是一个 Java 程序
  - 都是 Zygote 的后代 
