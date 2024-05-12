# 系统调用和 UNIX Shell

**本讲内容**：

- Shell
- xv6 shell 代码讲解

## (UNIX) Shell

### 为用户封装操作系统 API

> 我们需要一个 “用户能直接操作” 的程序管理操作系统对象。

 需要一个程序能<font color='red'>**协调多个应用程序**</font>

Shell: Kernel 的 “外壳”

- “与<font color='red'>**人类直接交互**</font>的第一个程序”

### The UNIX Shell

“终端” 时代的伟大设计

- “Command-line interface” (CLI) 的巅峰

<font color='red'>**Shell 是一门 “把用户指令翻译成系统调用” 的编程语言**</font>

> “Unix is user-friendly; it's just choosy about who its friends are.”

如果把 shell 理解成编程语言，“不好用” 好像也没什么毛病了

![UNIX 世界有很多历史遗留约定](pics/xkcd-tar.png)

### The Shell Programming Language

<font color='red'>**基于文本替换的快速工作流搭建**</font>

- 重定向: `cmd > file < file 2> /dev/null`
- 顺序结构: `cmd1; cmd2`, `cmd1 && cmd2`, `cmd1 || cmd2`
- 管道: `cmd1 | cmd2`
- 预处理: `$()`, `<()`
  - `$(ls | grep 'A')`
  - `vim <(ls)`：将 ls 的内容，作为 vim 的文件输入

- 变量/环境变量、控制流……

Job control：类比窗口管理器里的 “叉”、“最小化”

- jobs, fg, bg, wait
- (今天的 GUI 并没有比 CLI 多做太多事)

### 人工智能时代，我们为什么还要读手册？

今天的人工智能还是 “被动” 的

- 它还不能很好地告诉你，你应该去找什么
- Manual 是一个 complete source
  - 当然，AI 可以帮助你更快速地浏览手册、理解程序的行为

## 复刻经典

### A Zero-dependency UNIX Shell (from xv6)

Shell 是 Kernel 之外的 “壳”

- 它也是一个<font color='red'>**状态机**</font> (同 minimal.S)
  - 完全基于系统调用 API

随堂实验代码：移植了 xv6 的 shell

- 零库函数依赖 (`-ffreestanding` 编译、ld 链接)
- 可以作为最小 Linux 的 init 程序

支持的功能（命令需要绝对路径）

- 重定向/管道 `ls > a.txt`, `ls | wc -l`
- 后台执行 `ls &`
- 命令组合 `(echo a ; echo b) | wc -l`

### 如何阅读代码

strace

- 适当的分屏和过滤
- AI 使阅读文档的成本大幅降低

gdb

- AskGPT: How to debug a process that forks children processes in gdb?
  - AI 也可以帮你解释 (不用去淘文档了)
- 以及，定制的 visualization
  - 对于 Shell，我们应该显示什么？

### 理解管道

![img](pics/pipe.gif)

## 展望未来

### UNIX Shell: Traps and Pitfalls

在 “自然语言”、“机器语言” 和 “1970s 的算力” 之间达到优雅的平衡

- 平衡意味着并不总是完美

- 操作的 “优先级”？

  - `ls > a.txt | cat`
    - 我已经重定向给 a.txt 了，cat 是不是就收不到输入了？

  - `bash/zsh` 的行为是不同的
    - 所以脚本一般都是 `#!/bin/bash` 甚至 `#!/bin/sh` 保持兼容

### 另一个有趣的例子

![img](pics/sudo-sandwich.png)

```shell
$ echo hello > /etc/a.txt
bash: /etc/a.txt: Permission denied

# 重定向操作仍然是以当前用户的权限进行
# correct: sudo sh -c 'echo hello > /etc/a.txt'
$ sudo echo hello > /etc/a.txt
bash: /etc/a.txt: Permission denied
```

### 展望未来

> Open question: 我们能否从根本上**改变管理操作系统**的方式？

需求分析

- Fast Path: 简单任务
  - 尽可能快
  - 100% 准确
- Slow Path: 复杂任务
  - 任务描述本身就可能很长
  - 需要 “编程”

### 未来的 Shell

自然交互/脑机接口：心想事成

- Shell 就成为了一个应用程序的交互库
  - UNIX Shell 是 “自然语言”、“机器语言” 之间的边缘地带

系统管理与语言模型

- fish, zsh, [Warp](https://www.warp.dev/), ...
- Stackoverflow, tldr, [thef**k](https://github.com/nvbn/thefuck) (自动修复)
- Command palette of vscode (Ctrl-Shift-P)
- Predictable
  - 流程很快 (无需检查)，但可能犯傻
- Creative
  - 给你惊喜，但偶尔犯错