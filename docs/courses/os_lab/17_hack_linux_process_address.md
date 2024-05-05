# Linux 进程的地址空间

**本讲内容**：在我们的状态机模型中，进程的状态由 (M,R)两部分组成；其中 R (register) 是由体系结构决定的，而 M (memory) 则还有一些未解开的谜题：程序在初始时，并不是所有的内存都可以访问的，但我们却又的确可以申请很大的内存。这是如何实现的？

- 进程的地址空间
- mmap 系统调用
- 三类游戏外挂的实现原理
  - 金山游侠：内存修改
  - 按键精灵：GUI 事件发送
  - 变速齿轮：代码注入

## Linux 进程的地址空间

两个很基本 (但也很困难) 的问题

- 以下程序的 (可能) 输出是什么？

```c
printf("%p\n", main);
```

------

- **何种指针访问**不会引发 segmentation fault?

```c
char *p = random();
*p; // 什么时候访问合法？ 
```

### 查看进程的地址空间

`pmap (1) - report memory of a process`

- pmap 是通过访问 procfs (`/proc/`) 实现的，通过`strace`可以验证；

查看进程的地址空间

- 等程序运行起来后 (gdb)，使用 `pmap` 命令查看地址空间
- **地址空间是若干连续的 “内存段”**
  - “段” 的内存可以根据**权限**访问
  - 不在段内/违反权限的内存访问 触发 SIGSEGV

### 操作系统提供查看进程地址空间的机制

RTFM: `/proc/[pid]/maps` (man 5 proc)

进程地址空间中的每一段

- 地址 (范围) 和权限 (rwxsp)
- 对应的文件: offset, dev, inode, pathname

通过实验观察 address space 的变化

- 堆 (bss) 内存的大小：全局数组/malloc；
- 栈上的大数组 v.s. memory error

进程的内存示例：

- `vdso (7): vdso（virtual dynamic shared object）`
- `Virtual system calls(vsyscall)`: **只读的系统调用**也许可以不陷入内核执行。

无需陷入内核的系统调用

- 例子: time (2)：时间：内核维护秒级的时间 (所有进程映射同一个页面)
- 例子: gettimeofday (2)：[RTFSC](https://elixir.bootlin.com/linux/latest/source/lib/vdso/gettimeofday.c#L49) (非常聪明的实现)
- 更多示例：问 GPT 吧

```shell
0000555555554000 r--p     a.out
0000555555555000 r-xp     a.out               # 代码段
0000555555556000 r--p     a.out               
0000555555557000 r--p     a.out
0000555555558000 rw-p     a.out               # 数据段
00007ffff7dc1000 r--p     libc-2.31.so
00007ffff7de3000 r-xp     libc-2.31.so
00007ffff7f5b000 r--p     libc-2.31.so
00007ffff7fa9000 r--p     libc-2.31.so
00007ffff7fad000 rw-p     libc-2.31.so
00007ffff7faf000 rw-p     (这是什么？)
00007ffff7fcb000 r--p     [vvar] (这又是什么？)
00007ffff7fce000 r-xp     [vdso] (这叒是什么？)
00007ffff7fcf000 r--p     (省略相似的 ld-2.31.so)
00007ffffffde000 rw-p     [stack]
ffffffffff600000 --xp     [vsyscall] (这叕是什么？)
```

## 进程地址空间管理

### 地址空间 = 带访问权限的内存段

操作系统应该提供一个<font color='red'>**修改进程地址空间的系统调用**</font>

```c
// 映射
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);

// 修改映射权限
int mprotect(void *addr, size_t length, int prot);
```

------

本质：在状态机状态上增加/删除/修改一段可访问的内存

- mmap: 可以用来申请内存 (`MAP_ANONYMOUS`)，也可以把文件 “搬到” 进程地址空间中

### 把文件映射到进程地址空间？

它们的确好像没有什么区别

- 文件 = 字节序列 (操作系统中的对象)
- 内存 = 字节序列

ELF loader 用 mmap 非常容易实现

- 解析出要加载哪部分到内存，直接 mmap 就完了
- 我们的 loader 的确是这么做的 (strace)

### 使用 mmap

Example 1: 申请大量内存空间

- 瞬间完成内存分配
  - mmap/munmap 为 malloc/free 提供了机制
  - libc 的大 malloc 会直接调用一次 mmap 实现
- 不妨 strace/gdb 看一下

Example 2: Everything is a file

- 映射大文件、只访问其中的一小部分

```python
with open('/dev/sda', 'rb') as fp:
    mm = mmap.mmap(fp.fileno(),
                   prot=mmap.PROT_READ, length=128 << 30)
    hexdump.hexdump(mm[:512])
```

### Memory-Mapped File: 一致性

如果把页面映射到文件

- 修改什么时候生效？

  - 立即生效：那会造成巨大量的磁盘 I/O
  - unmap (进程终止) 时生效：好像又太迟了……
  
- 若干个映射到同一个文件的进程？

  - 共享一份内存？各自有本地的副本？
  

请查阅手册，看看操作系统是如何规定这些操作的行为的

- 例如阅读 `msync(2)`, `mmap()`
- 这才是操作系统真正的复杂性

> `mmap`的 flags 决定是 MAP_SHARED  还是 MAP_PRIVATE（copy-on-write）
>
> - MAP_SHARED  ：共享，对其它进程可见，更新会写回底层文件；
> - MAP_PRIVATE：私有的copy-on-write mappings，更新对其它进程不可见且不会写回到文件

## 🌶️ 入侵进程地址空间

### Hacking Address Spaces

进程 (*M*,*R* 状态机) 在 “无情执行指令机器” 上执行

- 状态机是一个封闭世界
- <font color='red'>**但如果允许一个进程对其他进程的地址空间有访问权**？</font>

一些入侵进程地址空间的例子

- 调试器 (gdb)：gdb 可以任意观测和修改程序的状态
- Profiler (perf)：合理的需求，操作系统就必须支持 → Ask GPT!

### 入侵进程地址空间 (0): 金手指

如果我们能直接物理劫持内存，不就都解决了吗？

- 听起来很离谱，但 “卡带机” 时代的确可以做到！

![img](pics/game-genie-connect.jpg)

Game Genie: 一个 Look-up Table (LUT)

- 当 CPU 读地址*a*时读到*x*，则替换为*y*
  - [NES Game Genie Technical Notes](https://tuxnes.sourceforge.net/gamegenie.html) ([专利](https://patents.google.com/patent/EP0402067A2/en), [How did it work?](https://www.howtogeek.com/706248/what-was-the-game-genie-cheat-device-and-how-did-it-work/))
  - 今天我们有 [Intel Processor Trace](https://perf.wiki.kernel.org/index.php/Perf_tools_support_for_Intel®_Processor_Trace)

### 入侵进程地址空间 (1): 金山游侠

<font color='red'>在进程的内存中找到代表 “金钱”、“生命” 的重要属性并且改掉</font>

![img](pics/knight.jpg)

包含非常贴心的 “游戏内呼叫” 功能

- 它就是游戏的 (阉割版) “调试器”
- 我们也可以在 Linux 中实现它 (man 5 proc)：通过直接读写 `/proc/$pid/mem`文件。

## 入侵进程地址空间 (2): 按键精灵

<font color='red'>大量重复固定的任务 (例如 2 秒 17 枪)</font>

![img](pics/ajjl.jpg)

这个简单，就是**给进程发送键盘/鼠标事件**

- 做个驱动 (可编程键盘/鼠标)
- 利用**操作系统/窗口管理器提供的 API**
  - [xdotool](https://github.com/jordansissel/xdotool) (我们用这玩意测试 vscode 的插件)
  - [evdev](https://www.kernel.org/doc/html/latest/input/input.html) (按键显示脚本；主播常用)

### 入侵进程地址空间 (3): 变速齿轮

<font color='red'>调整游戏的逻辑更新速度</font>

- 比如[某神秘公司](https://baike.baidu.com/item/台湾天堂鸟资讯有限公司/8443017)慢到难以忍受的跑图和战斗

![img](pics/speed-up.jpg)

本质：程序是状态机

- 除了 syscall，是不能感知时间的
- 只要 **“劫持” 和时间相关的 syscall**，就能改变程序对时间的认识
  - 原则上程序仍然可以用间接信息 “感知” 的 (就想表调慢了一样)
