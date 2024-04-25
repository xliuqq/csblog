# 操作系统上的进程

**本讲内容**：操作系统上的进程

- 线程、进程和操作系统
- UNIX/Linux 进程管理 API

## 线程、进程和操作系统

> 在 UNIX/Linux 系统内核完成初始化后，只有一个 init 进程被启动。
>
> - init 进程跟 `/usr/sbin/init`是不一样的
>
> 此后，操作系统内核就化身为了一个事件驱动的程序、状态机的管理者，仅在中断和系统调用发生时开始执行。

### ThreadOS 中的线程切换

> 课堂实验：中断时的寄存器和堆栈的查看。

计算机硬件也是状态机

- “共享的内存，容纳了多个状态机”
  - 共享的代码
  - 共享的全局变量
  - 启动代码的堆栈 (仅启动代码可用)
  - $T_1$ 的 Task (仅$T_1$和中断处理程序可用)
    - 堆栈 (中断时，寄存器保存在堆栈上)
  - $T_2$的Task (仅$T_2$和中断处理程序可用)
    - 堆栈 (中断时，寄存器保存在堆栈上)
- 状态迁移
  - 执行指令或响应中断

### 什么是操作系统？

<font color='red'>虚拟化：**操作系统同时保存多个状态机**</font>

- C 程序 = 状态机
  - 初始状态：`main(argc, argv)`
  - 状态迁移：指令执行
    - 包括特殊的系统调用指令 syscall
- 有一类特殊的系统调用可以管理状态机
  - `CreateProcess(exec_file)`
  - `TerminateProcess()`

从线程到进程：虚拟存储系统

- 通过虚拟内存实现每次 “拿出来一个执行”
- 中断后进入操作系统代码，“换一个执行”



## 复制状态机：fork()

### 状态机管理：创建状态机

> 如果要创建状态机，我们应该提供什么样的 API？

UNIX 的答案: fork

- 做一份状态机完整的复制 (内存、寄存器现场)

```
int fork();
```

- 立即复制状态机 (完整的内存)
  - 复制失败返回 -1 (errno)
  - 新创建进程返回 0
  - 执行 fork 的进程返回子进程的进程号

状态机是复制的，因此总能找到 “父子关系”

- 因此有了进程树 (pstree)

```
systemd-+-ModemManager---2*[{ModemManager}]
        |-NetworkManager---2*[{NetworkManager}]
        |-accounts-daemon---2*[{accounts-daemon}]
        |-at-spi-bus-laun-+-dbus-daemon
        |                 `-3*[{at-spi-bus-laun}]
        |-at-spi2-registr---2*[{at-spi2-registr}]
        |-atd
        |-avahi-daemon---avahi-daemon
        |-colord---2*[{colord}]
        ...
```

- 复制是全部复制，包括“缓冲区”的复制、全局变量等，因此需要小心处理。

### Fork Bomb

不停地创建进程，系统还是会挂掉的

```bash
:(){:|:&};:   # 刚才的一行版本

:() {         # 格式化一下
  : | : &
}; :

fork() {      # bash: 允许冒号作为标识符……
  fork | fork &
}; fork
```



## 重置状态机：execve()

### 状态机管理：重置状态机

execve：将当前进程**重置成一个可执行文件描述状态机的初始状态**

```c
int execve(const char *filename, char * const argv[], char * const envp[]);
```

- 执行名为 `filename` 的程序
- 允许对新状态机设置参数`argv(v)`和环境变量`envp(e)`
  - 刚好对应了 `main()` 的参数！
- execve 是<font color='red'>**唯一能够 “执行程序” 的系统调用**</font>
  - 因此也是**一切进程 strace 的第一个系统调用**

### 环境变量

“应用程序执行的环境”

- 使用`env`命令查看
  - `PATH`: 可执行文件搜索路径
  - `PWD`: 当前路径
  - `HOME`: home 目录
  - `DISPLAY`: 图形输出
  - `PS1`: shell 的提示符

`export`: 告诉 shell 在创建子进程时设置环境变量

## 终止状态机：exit()

### 状态机管理：销毁状态机

> 有了 fork, execve 我们就能自由执行任何程序了，最后只缺一个销毁状态机的函数！

`exit`立即摧毁状态机

```c
void _exit(int status)
```

- 销毁当前状态机，并允许有一个返回值
- 子进程终止会通知父进程 (后续课程解释)

### 结束程序执行的三种方法

- `exit(0)` -`stdlib.h` 中声明的 libc 函数：可以执行 `atexit`注册的 hook；
- `_exit`：glibc 的 syscall wrapper
  - 会调用syscall exit_group，终止进程（包括所有的线程） ；
  - 不会调用 `atexit`

- `syscall(SYS_exit, 0)`
  - 执行 “`exit`” 系统调用**终止当前线程**
  - 不会调用 `atexit`

## 课后习题/编程作业

### 1. 阅读材料

教科书 Operating Systems: Three Easy Pieces

- 第 3 章 - [Dialogue](./book_os_three_pieces/03-dialogue-virtualization.pdf)
- 第 4 章 - [Processes](./book_os_three_pieces/04-cpu-intro.pdf)
- 第 5 章 - [Process AP](./book_os_three_pieces/05-cpu-api.pdf)