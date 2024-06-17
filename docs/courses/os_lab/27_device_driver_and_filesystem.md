# 设备驱动程序与文件系统

**本讲内容**：很自然的问题是，如果操作系统上的程序想要访问设备，就必须把设备抽象成一个进程可以使用系统调用访问的操作系统对象，也就是设备驱动程序和文件系统：

- 设备驱动程序
- 文件系统 API

## 设备驱动程序

### I/O 设备的抽象

I/O 设备模型：一个能与 CPU 交换数据的接口/控制器

- 寄存器被映射到地址空间

![img](pics/canonical-device.png)

操作系统：设备也是操作系统中的对象

- 如何 “找到” 一个对象？
- 对象支持什么操作？

I/O 设备的主要功能：<font color='red'>**输入和输出**</font>

- “能够读 (read) 写 (write) 的<font color='red'>**字节序列 (流或数组)**</font>”
- 常见的设备都满足这个模型
  - 终端/串口 - 字节流
  - 打印机 - 字节流 (例如 PostScript 文件)
  - 硬盘 - 字节数组 (按块访问)
  - GPU - 字节流 (控制) + 字节数组 (显存)

操作系统：设备 = 支持各类操作的对象 (文件)

- read - 从设备某个指定的位置读出数据
- write - 向设备某个指定位置写入数据
- **ioctl - 读取/设置设备的状态**

### 设备驱动程序

把系统调用 (`read/write/ioctl/...`) “翻译” 成与设备寄存器的交互

- 就是一段普通的内核代码
- 但可能会睡眠 (例如 P 信号量，等待中断中的 V 操作唤醒)

例子：`/dev/` 中的对象

- `/dev/pts/[x]` - pseudo terminal
- `/dev/zero` - “零” 设备（输入都是0x00）
- `/dev/null` - “null” 设备（输出黑洞）
- `/dev/random`，`/dev/urandom` - 随机数生成器
  - 试一试：`head -c 512 [device] | xxd`
  - 以及观察它们的 strace：能看到访问设备的系统调用

### 例子: Lab 2 设备驱动

设备模型

- 简化的假设：设备从系统启动时就存在且不会消失
- 支持读/写两种操作：在无数据或数据未就绪时会等待 (P 操作)

```c
typedef struct devops {
  int (*init)(device_t *dev);
  int (*read) (device_t *dev, int offset, void *buf, int count);
  int (*write)(device_t *dev, int offset, void *buf, int count);
} devops_t;
```

I/O 设备看起来是个 “黑盒子”

- 写错任何代码就 simply “not work”
- 设备驱动：Linux 内核中最多也是质量最低的代码

### 字节流/字节序列抽象的缺点

设备不仅仅是数据，还有<font color='red'>**控制**</font>

- 尤其是设备的附加功能和配置
- 所有额外功能全部依赖 ioctl
  - `Arguments, returns, and semantics of ioctl() vary according to the device driver in question`

例子

- 打印机的打印质量/进纸/双面控制、卡纸、清洁、自动装订……
  - 一台几十万的打印机可不是那么简单 😂
- 键盘的跑马灯、重复速度、宏编程……
- 磁盘的健康状况、缓存控制……

### 例子：终端

“字节流” 以内的功能

- ANSI Escape Code

“字节流” 以外的功能

- `stty -a`
  - 终端大小怎么知道？终端大小变化又怎么知道？
- `isatty (3), termios (3)`
  - 大部分都是 ioctl 实现的

## Linux 设备驱动

### Nuclear Launcher

如何在 Linux 中为我们的 “核弹发射器” 编写一个设备驱动程序？

![img](./pics/nuke-launch.gif)

内核模块：一段可以被内核动态加载执行的代码

- Everything is a file
  - 设备驱动就是实现了`struct file_operations`的对象
    - 把文件操作翻译成设备控制协议
    - 调用到设备实现的 `file_operations`

### file_operations

```c
struct file_operations {
  struct module *owner;
  loff_t (*llseek) (struct file *, loff_t, int);
  ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
  ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
  int (*mmap) (struct file *, struct vm_area_struct *);
  unsigned long mmap_supported_flags;
  int (*open) (struct inode *, struct file *);
  int (*release) (struct inode *, struct file *);
  int (*flush) (struct file *, fl_owner_t id);
  int (*fsync) (struct file *, loff_t, loff_t, int datasync);
  int (*lock) (struct file *, int, struct file_lock *);
  ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
  long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
  long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
  int (*flock) (struct file *, int, struct file_lock *);
  ...
```

**为什么有两个 ioctl?**

```c
long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
```

- `unlocked_ioctl`: BKL (Big Kernel Lock) 时代的遗产
  - 单处理器时代只有 `ioctl`
  - 之后引入了 BKL, `ioctl` 执行时默认持有 BKL
  - (2.6.11) 高性能的驱动可以通过 `unlocked_ioctl` 避免锁
  - (2.6.36) `ioctl` 从 `struct file_operations` 中移除
- `compact_ioctl`: 机器字长的兼容性
  - 32-bit 程序在 64-bit 系统上可以 ioctl
  - 此时应用程序和操作系统对 ioctl 数据结构的解读可能不同 (tty)
  - (调用此兼容模式)

### 存储设备的抽象

磁盘 (存储设备) 的访问特性

1. 以**数据块 (block)** 为单位访问
   - 传输有 “最小单元”，不支持任意随机访问
   - 最佳的传输模式与设备相关 (HDD v.s. SSD)
2. 大吞吐量
   - 使用 DMA 传送数据
3. <font color='red'>**应用程序不直接访问**</font>
   - 访问者通常是文件系统 (维护磁盘上的数据结构)
   - 大量并发的访问 (操作系统中的进程都要访问文件系统)

### Linux Block I/O Layer

文件系统和磁盘设备之间的接口

- `bread` (读一块), `bwrite `(写一块), `bflush` (等待过往写入落盘)

![linux-bio](pics/linux-bio.png)

## 存储设备的虚拟化

### 文件系统：实现设备在应用程序之间的共享

磁盘中存储的数据

- 程序数据
  - 可执行文件和动态链接库
  - 应用数据 (高清图片、过场动画、3D 模型……)
- 用户数据
  - 文档、下载、截图
- 系统数据
  - Manpages
  - 配置文件 (/etc)

<font color='red'>**字节序列并不是磁盘的好抽象**</font>

- 让所有应用共享磁盘？一个程序 bug 操作系统就没了

### 文件系统：磁盘的虚拟化

文件系统：设计目标

1. 提供合理的 API 使多个应用程序能共享数据
2. 提供一定的隔离，使恶意/出错程序的伤害不能任意扩大

“存储设备 (字节序列) 的虚拟化”

- 磁盘 (I/O 设备) = 一个可以读/写的字节序列
- <font color='red'>**虚拟磁盘 (文件) = 一个可以读/写的动态字节序列**</font>
  - 命名管理：虚拟磁盘的名称、检索和遍历
  - 数据管理：`std::vector<char>` (随机读写/resize)

### 文件 = 虚拟磁盘

文件：虚拟的磁盘

- 磁盘是一个 “字节序列”
- 支持读/写操作

文件描述符：进程访问文件 (操作系统对象) 的 “指针”

- 通过 `open/pipe` 获得
- 通过 `close` 释放
- 通过 `dup/dup2` 复制
- `fork` 时继承

### mmap 和文件

```c
void *mmap(
  void *addr, size_t length, int prot, int flags,
  int fd, off_t offset // 映射 fd 文件
); 
```

文件是 “虚拟磁盘”

- 把磁盘的一部分映射到地址空间，再自然不过了

小问题：映射的长度超过文件大小会发生什么？

- (RTFM, “Errors” section):`SIGBUS`...
  - bus error 的常见来源 (M5)
  - ftruncate 可以改变文件大小

### 文件访问的游标 (偏移量)

文件的读写自带 “游标”，这样就不用每次都指定文件读/写到哪里了

- 方便了程序员顺序访问文件

例子

- `read(fd, buf, 512);` - 第一个 512 字节
- `read(fd, buf, 512);` - 第二个 512 字节
- `lseek(fd, -1, SEEK_END);` - 最后一个字节
  - so far, so good

### 偏移量管理：没那么简单

`mmap, lseek, ftruncate` 互相交互的情况

- 初始时文件大小为 0
  1. mmap (`length` = 2 MiB)
  2. lseek to 3 MiB (`SEEK_SET`)
  3. ftruncate to 1 MiB

在任何时刻，写入数据的行为是什么？

- blog posts 不会告诉你全部
- RTFM & 做实验！

> <font color='red'>**文件描述符在 fork 时会被子进程继承。**</font>

父子进程应该共用偏移量，还是应该各自持有偏移量？

- 这决定了 `offset` 存储在哪里

考虑应用场景：**父子进程同时写入文件**

- 各自持有偏移量 → 父子进程需要协调偏移量的竞争
  - (race condition)
- 共享偏移量 → 操作系统管理偏移量
  - 虽然仍然共享，但操作系统保证 `write` 的原子性 ✅

### 偏移量管理：行为

操作系统的每一个 API 都可能和其他 API 有交互 😂

1. `open` 时，获得一个独立的 offset
2. `dup` 时，两个文件描述符共享 offset
3. `fork` 时，父子进程共享 offset
4. `execve` 时文件描述符不变
5. `O_APPEND`方式打开的文件，偏移量永远在最后 (无论是否 fork)
   - modification of the file offset and the write operation are performed as a single **atomic** step

这也是 fork 被批评的一个原因

- (在当时) 好的设计可能成为系统演化过程中的包袱
  - 今天的 fork 可谓是 “补丁满满”；[A `fork()` in the road](https://dl.acm.org/doi/10.1145/3317550.3321435)

## 目录树管理

### 利用信息的局部性组织虚拟磁盘

> 信息的局部性：将虚拟磁盘 (文件) 组织成层次结构

目录树

- 逻辑相关的数据存放在相近的目录

```
.
└── 学习资料
    ├── .学习资料(隐藏)
    ├── 问题求解1
    ├── 问题求解2
    ├── 问题求解3
    ├── 问题求解4
    └── 操作系统
```

### 文件系统的 “根”

树总得有个根结点

- Windows: 每个设备(驱动器) 是一棵树
  - C:\ “C 盘根目录”
    - `C:\Program Files\`, `C:\Windows`, `C:\Users`, ...
  - 优盘分配给新的盘符
    - 为什么没有 `A:\`, `B:\`? => 软盘
    - 简单、粗暴、方便，但 `game.iso` 一度非常麻烦……

- UNIX/Linux
  - 只有一个根 / 
    - 第二个设备呢？
    - 优盘呢？？？

### 目录树的拼接

UNIX: 允许任意目录 “挂载 (mount)” 一个设备代表的目录树

- 非常灵活的设计
  - 可以把设备挂载到任何想要的位置
  - Linux 安装时的 “mount point”
    - `/`, `/home`, `/var` 可以是独立的磁盘设备

mount 系统调用

```c
int mount(const char *source, const char *target,
          const char *filesystemtype, unsigned long mountflags,
          const void *data);
```

- `mount /dev/sdb /mnt`(RTFM)
  - Linux mount 工具能自动检测文件系统 (busybox 不能)

### 真正的 Linux 启动流程

Linux-minimal 运行在 “initramfs” 模式

- Initial RAM file system
- 完整的文件系统
  - 可以包含设备驱动等任何文件
  - 但不具有 “持久化” 的能力

最小 “真正” Linux 的启动流程

```shell
export PATH=/bin
busybox mknod /dev/sda b 8 0
busybox mkdir -p /newroot
busybox mount -t ext2 /dev/sda /newroot
exec busybox switch_root /newroot/ /etc/init
```

通过 `pivot_root` (2) 实现根文件系统的切换

### 文件的挂载

文件的挂载引入了一个微妙的循环

- 文件 = 磁盘上的虚拟磁盘
- 挂载文件 = 在虚拟磁盘上虚拟出的虚拟磁盘

Linux 的处理方式

- 创建一个 loopback (回环) 设备
  - 设备驱动把设备的 `read/write` 翻译成文件的 `read/write`
- 观察 disk-img.tar.gz 的挂载
  - `lsblk` 查看系统中的 block devices (strace)
  - `strace` 观察挂载的流程
    - `ioctl(3, LOOP_CTL_GET_FREE)`
    - `ioctl(4, LOOP_SET_FD, 3)`

### [Filesystem Hierarchy Standard](http://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) (FHS)

> FHS enables *software and user* to predict the location of installed files and directories.

例子：macOS 是 UNIX 的内核 (BSD), 但不遵循 Linux FHS。

### 目录管理：创建/删除/遍历

- `mkdir`：创建一个目录
  - 可以设置访问权限
- `rmdir`：删除一个空目录
  - 没有 “递归删除” 的系统调用
    - (应用层能实现的，就不要在操作系统层实现)
    - `rm -rf` 会遍历目录，逐个删除 (试试 strace)
- `getdents`：返回 count 个目录项 (ls, find, tree 都使用这个)
  - 以点开头的目录会被系统调用返回，只是 ls 没有显示

合适的 API + 合适的编程语言

- [Globbing](https://www.gnu.org/software/libc/manual/html_node/Calling-Glob.html)

```python
from pathlib import Path

for f in Path('/proc').glob('*/status'):
    print(f.parts[-2], \
        (f.parent / 'cmdline').read_text() or '[kernel]')
```

### 硬 (hard) 链接

需求：系统中可能有同一个运行库的多个版本

- `libc-2.27.so`, `libc-2.26.so`, ...
- 还需要一个 “当前版本的 libc”
  - 程序需要链接 “`libc.so.6`”，能否避免文件的一份拷贝？

硬连接：允许一个文件被多个目录引用

- 目录中仅存储指向文件数据的指针
- 链接目录 ❌
- 跨文件系统 ❌

大部分 UNIX 文件系统所有文件都是硬连接 (`ls -i` 查看)

- 删除的系统调用称为 “unlink” (引用计数)

### 软 (symbolic) 链接

软链接：在文件里存储一个 “跳转提示”

- 软链接也是一个文件
  - 当引用这个文件时，去找另一个文件
  - 另一个文件的绝对/相对路径以文本形式存储在文件里
  - <font color='red'>**可以跨文件系统、可以链接目录**</font>、……
- 类似 “快捷方式”
  - 链接指向的位置**当前不存在也没关系**
  - `~/usb` → `/media/jyy-usb`
  - `~/Desktop` → `/mnt/c/Users/jyy/Desktop` (WSL)

`ln -s` 创建软链接：`symlink` 系统调用

### 软链接带来的麻烦

“任意链接” 允许创建任意有向图

- 允许多次间接链接
  - a → b → c (递归解析)
- 可以创建软连接的硬链接 (因为软链接也是文件)
  - `ls -i` 可以看到
- 允许成环
  - `find -L A | tr -d '/'`
  - 可以做成一个 “迷宫游戏”
    - ssh 进入游戏，进入名为 end 的目录胜利
    - 只允许 ls (-i), cd, pwd
  - **所有处理符号链接的程序 (tree, find, ...) 都要考虑递归的情况**
