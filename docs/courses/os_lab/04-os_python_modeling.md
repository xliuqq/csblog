# Python 建模操作系统

> 在中断或是 trap 指令后，通常由一段汇编代码将当前状态机 (执行流) 的寄存器保存到内存中，完成状态的 “封存”。

## 理解操作系统的新途径

计算机软件

- 状态机 (C/汇编)
  - 允许执行特殊指令 (syscall) 请求操作系统
  - 操作系统 = API + 对象

计算机硬件

- “无情执行指令的机器”
  - 从 CPU Reset 状态开始执行 Firmware 代码
  - 操作系统 = C 程序

状态和状态的迁移是可以 “画” 出来的！

- 理论上说，只需要两个 API
  - `dump_state()` - 获取当前程序状态
  - `single_step()` - 执行一步

 概念

- 应用程序 (高级语言状态机)
- 系统调用 (操作系统 API)
- 操作系统内部实现

通常的思路：真实的操作系统 + QEMU/NEMU 模拟器

- 我们的思路
  - 应用程序 = 纯粹计算的 Python 代码 + 系统调用
  - 操作系统 = Python 系统调用实现，有 “假想” 的 I/O 设备

## 操作系统 “玩具”：设计与实现

### API

四个 “系统调用” API

- **choose(xs)**: 返回 `xs` 中的一个随机选项
- **write(s)**: 输出字符串 `s`
- **spawn(fn)**: 创建一个可运行的状态机 `fn`
- **sched()**: 随机切换到任意状态机执行

除此之外，所有的代码都是确定 (deterministic) 的纯粹计算

- 允许使用 list, dict 等数据结构

**sys_sched()** 的难点：

- 封存当前状态机的状态
- 恢复另一个 “被封存” 状态机的执行

Python 语言机制：**Generator objects**

```python
def numbers():
    i = 0
    while True:
        # f'{i:b}' 将i转为2进制
        # send 中参数会替代 yield 赋值给 ret
        ret = yield f'{i:b}'  # “封存” 状态机状态
        i += ret
```

使用方法：

```python
n = numbers()  # 封存状态机初始状态
n.send(None)   # 恢复封存的状态，开始时只能发送 None，输出 ‘0’
n.send(0)      # 恢复封存的状态 (并传入返回值)，
```



## 建模操作系统

进程 + 线程 + 终端 + 存储 (崩溃一致性)

| 系统调用/Linux 对应          | 行为                            |
| :--------------------------- | :------------------------------ |
| sys_spawn(fn)/pthread_create | 创建从 fn 开始执行的线程        |
| sys_fork()/fork              | 创建当前状态机的完整复制        |
| sys_sched()/定时被动调用     | 切换到随机的线程/进程执行       |
| sys_choose(xs)/rand          | 返回一个 xs 中的随机的选择      |
| sys_write(s)/printf          | 向调试终端输出字符串 s          |
| sys_bread(k)/read            | 读取虚拟设磁盘块 �*k* 的数据    |
| sys_bwrite(k, v)/write       | 向虚拟磁盘块 �*k* 写入数据 �*v* |
| sys_sync()/sync              | 将所有向虚拟磁盘的数据写入落盘  |
| sys_crash()/长按电源按键     | 模拟系统崩溃                    |

**模型的简化**

被动进程/线程切换

- 实际程序随时都可能被动调用 `sys_sched()` 切换

只有一个终端

- 没有 `read()` (用 choose 替代 “允许读到任意值”)

磁盘是一个 `dict`

- 把任意 key 映射到任意 value
- 实际的磁盘
  - key 为整数
  - value 是固定大小 (例如 4KB) 的数据
  - 二者在某种程度上是可以 “互相转换” 的

**模型实现**

- 原理与刚才的 “最小操作系统玩具” 类似
  - [mosaic.py](https://gitee.com/oscsc/oslabcode/tree/master/ch04-os_python_modeling/mosaic/mosaic.py) - 500 行建模操作系统
  - 进程/线程都是 Generator Object
  - 共享内存用 heap 变量访问
    - 线程会得到共享 heap 的指针
    - 进程会得到一个独立的 heap clone
- 输出程序运行的 “状态图”
  - JSON Object
  - Vertices: 线程/进程、内存快照、设备历史输出
  - Edges: 系统调用
    - 操作系统就是 “状态机的管理者”