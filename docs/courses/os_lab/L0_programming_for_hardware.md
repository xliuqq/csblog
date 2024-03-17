# L0: 为计算机硬件编程

> [L0: 为计算机硬件编程 (jyywiki.cn)](https://jyywiki.cn/OS/2023/labs/L0.html)

## 1. 背景

很多 Tutorial 把硬件的细节 (例如 x86 的 GDT, IDT, TSS, ...) 和操作系统实现捆绑在一起，就显得更难上手。

**操作系统和硬件之间的关系被夸大了**：

- 硬件的确提供了必要的机制使我们能实现诸如进程/线程切换，但并不意味着除非我们知道有关硬件指令的一切才能编写操作系统；
- 操作系统**在 99.99% 的时候都是普通的 C 程序**，只是偶尔会干一些 “出格” 的事情；
- 当然会在适当的时候引入这些骚 (黑) 操 (科) 作 (技)，以及它们是如何借助硬件指令和机制实现；

## 2. 实验描述

### 2.1. 阅读 AbstractMachine 文档

> [阅读 AbstractMachine 的文档](https://jyywiki.cn/AbstractMachine/index.html)

建立了一些基本概念后，就可以试着从样例代码出发开始编程

- 实验框架中直接包含了 AbstractMachine 的代码并在 Makefile 中完成了配置 (RTFSC)，无需额外配置/下载

### 2.1.实现 AbstractMachine 中 klib 中缺失的函数

AbstractMachine 框架代码中包含一个基础运行库的框架 klib，其中包含了方便你编写 “bare-metal” 程序的系列库函数



### 2.2. 在 AbstractMachine 中显示一张图片



## 4. 实验指南

### 4.1. 实现库函数



### 4.2. 访问 I/O 设备



### 4.3. 绘制一个图片

