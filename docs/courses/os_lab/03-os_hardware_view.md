# 硬件视角的操作系统

> 操作系统有三条主线：“软件 (应用)”、“硬件 (计算机)”、“操作系统 (软件直接访问硬件带来麻烦太多而引入的中间件)”。第二章讲解操作系统上的应用程序的本质 (状态机)。而程序最终是运行在计算机硬件上的，本章讲解什么是计算机硬件，以及如何为计算机硬件编程。

**本讲内容**：计算机硬件的状态机模型；回答以下问题：

- 什么是计算机硬件？
- 计算机硬件和程序员之间是如何约定的？
- 听说操作系统也是程序。那到底是鸡生蛋还是蛋生鸡？



## 计算机硬件的状态机模型

**计算机硬件 = 数字电路**，数字电路模拟器 (Logisim)

- 基本构件：wire, reg, NAND
- 每一个时钟周期
  - 先计算 wire 的值
  - 在周期结束时把值锁存至 reg

<font color=red>不仅是程序，整个**计算机系统也是一个状态机**</font>

- 状态：内存和寄存器数值
- 初始状态：手册规定 (**CPU Reset**)
- 状态迁移
  - 任意选择一个处理器 cpu
  - 响应处理器外部中断
  - 从 cpu.PC 取指令执行



## 硬件与程序员的约定

Bare-metal 与厂商的约定

- **CPU Reset** 后的状态 (即寄存器值，包括 PC)
  - 厂商自由处理这个地址上的值（在 PC 的位置，写入代码）
  - Memory-mapped I/O
  - [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://software.intel.com/en-us/articles/intel-sdm), Volume 3A/3B 的规定

厂商为操作系统开发者提供 Firmware（驱动和固件）

- 管理硬件和系统配置
- <font color=red>把存储设备上的代码加载到内存</font>
  - 例如存储介质上的第二级 loader (加载器)
  - 或者直接加载操作系统 (嵌入式系统)

Firmware 负责加载操作系统

- 开发板（物理）：直接把加载器写入 ROM
- QEMU（模拟）：`-kernel` 可以绕过 Firmware 直接加载内核 ([RTFM](https://www.qemu.org/docs/master/system/invocation.html#hxtool-8))



Firmware：[BIOS vs. UEFI](https://www.zhihu.com/question/21672895)

- **UEFI(Unified Extensible Firmware Interface)**：应对越来越多的硬件，各种驱动程序，解决BIOS逐渐**碎片化**的问题；

<img src="pics/UEFI-booting-seq.png" alt="img" style="zoom:50%;" />

**Legacy BIOS的约定**，提供机制，将程序员的代码载入内存

- Legacy BIOS 把第一个可引导设备（A盘、B盘、C盘等）的第一个 512 字节加载到物理内存的 `7c00` 位置

**UEFI 上的操作系统加载**

- 磁盘必须按 GPT (GUID Partition Table) 方式格式化
- 预留一个 FAT32 分区 (lsblk/fdisk 可以看到)
- Firmware 能够加载任意大小的 PE 可执行文件 `.efi`
  - 没有 legacy boot 512 字节限制
  - EFI 应用可以返回 firmware

## 模拟器

> 有没有可能我们真的去看从 CPU Reset 以后每一条指令的执行？

<font color=red>计算机系统公理：你想到的就一定有人做到</font>

- 模拟方案：[Qemu 模拟器](../../cloud/virtualize/README.md#Qemu)
- 真机方案：JTAG (Joint Test Action Group) debugger



## Bare-metal 上的 C 代码

为了让下列程序能够 “运行起来”：

```c
int main() {
  printf("Hello, World\n");
}
```

------

需要准备

- MBR 上的 “启动加载器” (Boot Loader)
- 我通过编译器控制 C 程序的行为
  - 静态链接/PIC (位置无关代码)
  - Freestanding (不使用任何标准库)
  - 自己手工实现库函数 (putch, printf, ...)

### AbstractMachine: 抽象计算机

提供**运行 C 程序的框架代码和库**。裸机上的 C 语言运行环境，提供 5 组 (15 个) 主要 API，可以实现各类系统软件 (如操作系统)：

- (TRM) `putch`/`halt` - 最基础的计算、显示和停机
- (IOE) `ioe_read/ioe_write` - I/O 设备管理
- (CTE) `ienabled`/`iset`/`yield`/`kcontext` - 中断和异常
- (VME) `protect`/`unprotect`/`map`/`ucontext` - 虚存管理
- (MPE) `cpu_count`/`cpu_current`/`atomic_xchg` - 多处理器



## 实验

[M1: 打印进程树 (pstree)](https://jyywiki.cn/OS/2023/labs/M1)



[L0: 为计算机硬件编程 ](https://jyywiki.cn/OS/2023/labs/L0)
