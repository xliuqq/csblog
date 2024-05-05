# 操作系统实验生存指南

**本讲内容**：

- 编程中的一些细节
- 调试工具的正确使用方法

## 写 “好” 的程序

> 我反对编程自学，当然前提是你的老师会 “编程”。		——jyy

### 精益求精

代码规范：Guidelines

- [Google](https://google.github.io/styleguide/cppguide.html), [GNU](https://www.gnu.org/prep/standards/html_node/Writing-C.html), [CERT-C](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)
- 变量起哪些名字：
  - AskGPT: Is it a good idea to use variable name "rc" for a variable holding a 0/-1 return value of an API/system call in C?

Programming **for fun**

- [The International Obfuscated C Code Contest](https://www.ioccc.org/)
  - 写出绝对不可读，但又绝对可用的代码
  - [一个程序的诞生](https://jyywiki.cn/pages/OS/img/ioccc-spoiler.html)

### Timothy Roscoe's Keynote on OSDI/ATC'21

大多数 OS 研究社区成员都犯了 ignorance 和 denial 两个问题：

- ignorance：尽管这些现代硬件的文档都非常完善，但我们中很多人还是根本不知道现代硬件是怎么样的，还觉得它们不过是花哨些的 VAX ；

- 很多人囿于自己的舒适区，专注于 Linux 擅长的硬件假设上，觉得其他的东西都是些设备驱动之类的东西，不归他们这些做“OS”的人管，他们只需要知道 Linux 是怎样的就行；![img](pics/roscoe-keynote.png)

## 用好工具

### 回到笨拙的 info inferiors 和 pmap

你们一定不会愿意用的：就像你不希望用汇编写代码

- AskGPT: Can I print process memory maps inside gdb?
- AskGPT: Can I visualize a process-internal data structure in gdb?
  - (GPT 没给我 “惊喜”，但我没有他那么好的记忆读取能力)

### 计算机系统公理

> “三省吾身：滚去编程了吗？写测试用例了吗？回看自己的工作流程了吗？”

1. 机器永远是对的
   - (ICS PA/OS Labs: 怕是不用多说了)
2. 未测代码永远是错的
   - (ICS PA/OS Labs: 怕是不用多说了)
3. <font color='red'>**让你感到不适的 tedious 工作，一定有办法提高效率**</font>
   - 推论：我们应该分辨出什么工作是 tedious 的

### 更进一步：连接两个世界

程序 = 计算机系统 = 状态机

- 调试器的本质是 “检查状态”
- 我们能不能用自己想要的方式去检查状态？
  - 例如，像 model checker 那样把一个链表绘制出来？
- AskGPT: How to use Python to parse and check/visualize C/C++ program state in GDB?
  - GPT-4 甚至给出了正确的[文档 URL](https://sourceware.org/gdb/onlinedocs/gdb/Python-API.html)

 然后我们就可以做任何事了

- [TUI Window API](https://sourceware.org/gdb/onlinedocs/gdb/TUI-Windows-In-Python.html#TUI-Windows-In-Python)
- 甚至，我们可以做另一个 debugging front-end：[gdbgui](https://www.gdbgui.com/)

### 调试困难的根本原因

我们只能检查一个瞬间的状态

- Reverse debugging 的成本还是太高了
- 而且 time-travel 也很难操作
  - 如果我们能精简状态就好了！
  - 精简状态不就行了吗……

自己动手做工具

- 调试 thread-os（随堂实验的代码）

### 我们需要的：想象力

The UNIX Philosophy

- Keep it simple, stupid
- 把 gdb 状态导出 serialize 到文件/管道中
- 由另一个程序负责处理
  - 就像我们的 interactive state space explorer

assertions 也不一定要写在 C 代码里

- 导出的状态可以做任何 trace analysis