# L1: 物理内存管理 (pmm)

## 1. 背景

多处理器上的物理内存管理，具体完成物理内存的分配和回收。

> #### 谨慎使用动态内存分配
>
> 如果你没有写过相当规模的程序，在对象的种类增加时，内存管理会带来相当麻烦。 **对于对迷你自制操作系统来说，用固定大小的数组管理对象是一个好办法** (xv6 中就经常这么做)，分类管理的对象 (而不是混在一个全局的 “池子” 里) 对调试更友好，而且对象的大小通常有限，如果存在泄漏也很容易触发。

## 2. 实验描述

### 2.1. 实现多处理器安全的内存分配和回收

在 AbstractMachine 启动后，`[heap.start, heap.end)` 会给出一段可用的物理内存 (堆区)。你需要在此基础上实现允许多个处理器**并发**地申请或释放内存的分配器，包括以下两个函数：

```c
static void *kalloc(size_t size) {
  // 内存分配
}

static void kfree(void *ptr) {
  // 内存释放
}
```

具体来说，这个实验维护一个数据结构 (抽象数据类型)，维护一个不相交区间的集合 (堆区)

$H=\{[l_0,r_0), [l_1,r_1), ..., [l_n,r_n)\}$

初始时，堆区为空。假设当前堆区为 �*H*，`heap.start` =� =*L*, `heap.end` =� =*R*，你需要实现的操作有：

- $kalloc (s)$ 分配 *s* 字节的内存 $[l,r)$满足：
  1. 分配发生在堆区中 $L \le l \le r \le R $
  2. 分配的内存不与已分配内存重叠 $\forall [l_i,r_i] \in H. [l,r) \cap [l_i, r_i) = \empty]$
  3. 分配后新的堆区 $H^{'} = H \cup \{[l,r)\}$
  4. 返回分配区间的左端点 $l$

- $kfree(l)$删除一个已有区间$[l,r]\in H$，得到新的堆区 $H^{'} = H \textbackslash {[l,r)}$
  - 当 $l$ 不是 $H$ 中任何一个区间左节点时，产生 undefined behavior ——即 kfree 的调用者有义务保证一定释放一个已分配的空间。

作为**计算机系统软件基础设施**的要求：

- <font color='red'>**对于大小为 s 的内存分配请求，返回的内存地址必须对齐到$2^i$**</font>，其中 $i$ 是最小的正整数满足$2^i \ge s$。例如，4 KiB 的内存分配请求返回的地址必须是 4,096 的整数倍 (页面边界)。例如，分配 17 字节内存返回的地址必须是 32 的整数倍。这么要求其实简化了大家的实现。
- 在你的分配算法不能找到足够的内存可以分配时，返回 NULL(0)。
  - 受分配算法的局限，可能系统中仍然有空闲的内存，但形成了碎片的内存，或是你的算法不能找到它从而分配失败。这是允许的；但在系统内存压力很小的时候依然频繁分配失败，可能导致 hard test failure。
- 由于我们在自制的操作系统内核中使用，你<font color='red'>**可以直接拒绝超过 16 MiB 的内存分配**</font>。
- 不必初始化返回的内存，当然，对返回的内存赋上初始值 (例如零) 也许是个不错的主意。
- <font color='red'>**允许多处理器并行地调用 kalloc/kfree**</font>：
  - 不同的处理器可能同时执行 kalloc 分配大小不同的内存；
  - 不同的处理器可能同时执行 kfree 释放内存；
  - 在一个处理器分配的内存，可能在另一个处理器上释放；但我们保证我们的测试代码中不存在数据竞争；
  - 在 kalloc/kfree 实现正确的前提下，尽可能使不同处理器上的内存分配能够并行。如果你代码的性能过低 (例如用全局的锁保护空闲链表，并用遍历链表的方式寻找可用的内存)，可能会导致 hard test failure。

### 2.2. 性能优化 (Hard Tests)

多处理器上的内存分配和释放带来额外的挑战：

1. 我们希望保证分配的正确性。这意味着用一把大锁保护一个串行数据结构 (例如算法导论上的 “区间树” 就能在 *O*(log*n*) 的时间内完成分配和释放操作) 就能正确实现多处理器上的内存分配。
2. 我们希望保证分配在多处理器上的性能和伸缩性 (scalability)。这意味着不同的处理器能够并行地申请内存，尽可能减少因为同时申请而发生一个处理器等另一个处理器的情况。

上面两点需求是矛盾的，这也是本次实验的重点挑战。在这个实验里，我们通过 “system” 的方法解决问题，而不是巧妙的算法/数据结构——这不意味着算法/数据结构在计算机系统中没有应用，只是在这个问题上并不非常必要。

### 2.3. 实现 AbstractMachine 中 klib 中缺失的函数

在 Lab0 中，我们已经提出了这个实验要求。这个实验建议大家实现 klib 里缺失的函数——没有 printf, sprintf 等函数，你根本就是在用汇编语言写操作系统，你无法活到最后。我们不会检查大家的 klib 实现，但也请大家不要抄袭——虽然互联网上有很多代码，但自己写一遍依然是掌握这些库函数原理的最好方式。

## 3. 正确性标准

### 3.1. 测试用例说明

为了强迫大家使用动态的方式管理大小不同的堆区，我们规定你使用的所有静态内存 (64-bit 模式下的代码、数据、bss) 不能超过 1 MiB。我们将会在运行之前对内核使用 size 命令进行检查，并拒绝过大的 kernel。管理堆区的数据结构应当在堆区中而不是静态区中进行分配。

我们会使用来自类似于真实操作系统系统内核的 workload (多处理器) 来测试大家的程序。系统/workload 的一些假设：

- 不超过 8 个处理器、不少于 64 MiB、不多于 4 GiB 内存的堆区；
- 大量、频繁的小内存分配/释放；其中绝大部分不超过 128 字节；
- 较为频繁的，以物理页面大小为单位的分配/释放 (4 KiB)；
- 非常罕见的大内存分配。

我们预期你的 kalloc/kfree 具有足够的 robustness，足以承载操作系统内核的实现:

1. Safety: 我们预期你的 kalloc/kfree 实现是正确的。一旦以下情况发生，将被立即被 Online Judge 判定为错误：
   - 虚拟机重启、crash、测试代码中的 assert fail 等 (通常由 undefined behavior 触发，例如数据竞争)
   - kalloc 的返回值不满足 API 规约，例如分配的内存不位于堆区、错误的内存对齐等
   - kalloc 返回一段尚未被 kfree 的内存
2. Liveness: 我们还需要系统有一定的 liveness，即系统中有充足剩余内存时，内存分配不应过于频繁的频繁失败。这是为了防止你为了 safety 进行一些 “投机取巧”：对所有 kalloc 都返回 `NULL`，safety trivially 被满足。
   - liveness 并不是一个 “硬性” 的要求。在合理的 workload 上，你程序的表现只要不比一个 (实现得非常一般) 的 baseline 差太多 (你依然允许在某些你认为无法满足的 kalloc 上返回 `NULL`)，就会被判定为正确

与 L0 类似，我们会链接我们 klib 的参考实现，因此你不必担心你的 klib 有些许问题；但你要保持 klib 库函数的行为与 libc 一致。注意 native 会链接系统本地的 glibc (而非 klib)。

### 3.2. Online Judge 运行测试代码的方式

我们会替换 `framework/main.c` 的代码，但保持的 `src/` 目录下的文件不变。因此，你自己的测试用例 (在 `os_run()` 中) 可以不必修改。测试时，我们会改变 `mpe_init` 的入口 (不再从 `os->run` 开始执行，而是直接运行测试的 workload)。因此你必须在 `os->init()` 中完成所有必要的初始化，例如调用 `pmm->init()`。

以下是一个我们测试代码的例子，它会不断生成随机的 kalloc/kfree 请求 (你可以假设我们的生成器总是生成合法的请求，例如不会发生 double-free 等)，并且检查 kalloc/kfree 返回结果的合法性；当检测到问题后会 assert fail 退出，然后你可能会得到 “Runtime Error”。

```c
#include <common.h>
#include <klib.h>

enum ops { OP_ALLOC = 1, OP_FREE };
struct malloc_op {
  enum ops type;
  union { size_t sz; void *addr; };
};

void stress_test() {
  while (1) {
    struct malloc_op op = random_op();
    switch (op.type) {
      case OP_ALLOC: alloc_check(pmm->alloc(op.sz), op.sz); break;
      case OP_FREE:  free(op.addr); break;
    }
  }
}

int main() {
  os->init();
  mpe_init(stress_test);
}
```

## 4. 实验指南

### 4.1. 代码组织与运行

实验框架代码由三个目录组成：

```
.
├── framework  -> 框架代码；可以在本地修改，但 Online Judge 评测时会被替换成我们的版本
│   ├── kernel.h
│   └── main.c
├── include    -> 头文件；可以自由修改/创建文件 (Online Judge 会复制)
│   └── common.h
├── Makefile
└── src        -> 源文件；可以自由修改/创建文件 (Online Judge 会复制)
    ├── os.c
    └── pmm.c
```

阅读 make 命令查看 make的输出：

- 可以 `:%s/^/\r` 在命令之间插入空行；`:%s/ /\r /g` 将命令的参数缩进排版

```c
$ make -nB # RTFM!
... (完全不可读的输出)
```

### 4.2. 框架代码导读

框架代码很短，它的 `main` 函数首先执行 `os` 的初始化，然后启动多个处理器，每个处理器都跳转到 `os->run` 执行：

```c
int main() {
  os->init();
  mpe_init(os->run);
  return 1;
}
```

`os` 是一个操作系统的 “模块”，可以看成是我们用 C 实现的面向对象编程，能增加代码的可读性。整个框架代码中唯一有些难以理解的部分就是模块的声明和定义 (`framework/kernel.h`)：

```c
#define MODULE(mod) \
  typedef struct mod_##mod##_t mod_##mod##_t; \
  extern mod_##mod##_t *mod; \
  struct mod_##mod##_t

#define MODULE_DEF(mod) \
  extern mod_##mod##_t __##mod##_obj; \
  mod_##mod##_t *mod = &__##mod##_obj; \
  mod_##mod##_t __##mod##_obj
```

我们用 `MODULE` 声明一个模块，用 `MODULE_DEF` 实际定义它。

这个宏的视觉效果很差，为了阅读它，大家不妨可以在刚才看到的编译命令中把 `-c -o ...` 替换成 `-E`，就得到了预编译后的代码：

```c
typedef struct mod_os_t mod_os_t;
extern mod_os_t *os;
struct mod_os_t {
  void (*init)();
  void (*run)();
};

...

extern mod_os_t __os_obj;
mod_os_t *os = &__os_obj;
mod_os_t __os_obj = {
  .init = os_init,
  .run = os_run,
};
```

然后你就可以对照着阅读预编译的代码 (用对了方法，也就没有那么难了)，只需要记住以下两点：

1. 宏是字面替换，`MODULE(mod)` 中所有的 “`mod`” 都会被替换掉；
2. `##` 是 C 语言用来拼接标识符的机制，`sys##tem` 将会得到 `system`;

如果你阅读上述代码感到障碍，你可能需要补充：(1) `typedef` 类型别名定义, (2) `extern` 关键字, (3) C11 结构体初始化的知识。另外，下划线 `_` 也可以是标识符的一部分。

随着实验的进展，你会发现模块机制清晰地勾勒出了操作系统中各个部分以及它们之间的交互，能够帮助大家更好地理解操作系统的实现原理。

### 4.3. 框架代码的执行

> #### 好消息：实验做出的简化
>
> 在这个实验中，我们只启动了 AbstractMachine 中的 TRM 和 MPE，因此你不必考虑中断和多处理器同时带来的诸多麻烦 (下一个实验才开始考虑)。你不妨把每个处理器想象成一个线程。

操作系统内核，目前只有两个函数：

- `os->init()` 完成操作系统所有部分的初始化。`os->init()` 运行在系统启动后的第一个处理器上，中断处于关闭状态；此时系统中的其他处理器尚未被启动。因此在 `os->init` 的实现中，你完全不必考虑数据竞争等多处理器上的问题。
- `os->run()` 是所有处理器的入口，在初始化完成后，框架代码调用 `_mpe_init(os->run)` 启动所有处理器执行。框架代码中，`os->run` 只是打印 Hello World 之后就开始死循环；你之后可以在 `os->run` 中添加各种测试代码。

所以就想象成是你的 `os->run()` 就是 `threads.h` 里创建的一个线程，仅此而已！

### 4.4. 实现 kalloc/kfree

实现主要在 pmm (physical memory management) 模块：

```c
MODULE(pmm) {
  void  (*init)();
  void *(*alloc)(size_t size);
  void  (*free)(void *ptr);
};
```

模块包含三个函数指针：

- `pmm->init()` 初始化 pmm 模块，它应当在多处理器启动前 (`os->init()` 中) 调用。你会在这里完成数据结构、锁的初始化等；
- `pmm->alloc()` 对应实验要求中的 kalloc；
- `pmm->free()` 对应实验要求中的 kfree。

框架代码中包含了 “空的” pmm 实现，它对任何内存分配的请求都返回失败，因此它满足 safety，但完全没有 liveness，仿佛堆区的大小为零：

```c
static void *kalloc(size_t size) { return NULL; }
static void kfree(void *ptr) { } 
static void pmm_init() { } 
MODULE_DEF(pmm) = {
  .init  = pmm_init,
  .alloc = kalloc,
  .free  = kfree,
};
```

> #### 思考题：static 函数
>
> 为什么 kalloc/kfree/pmm_init 声明成了 `static`？我把 `static` 去掉依然可以编译呀？
>
> 这是一个好的编程习惯，在 C 这样缺少 module/package/namespace 机制的编程语言上有效减少意外发生。

在讲解 “并发数据结构” 时讲解了 kalloc/free 的算法和实现。你应该阅读：

- 教科书第 29 章 (并发数据结构)，学习如何对数据结构进行并发控制；
- 教科书第 17 章 (空闲空间管理)，学习如何管理物理内存；
- 互联网上的其他资料。如果你希望了解现代 malloc 实现，你可以参考来自两个代码巨头的设计：Google 的 [tcmalloc](https://google.github.io/tcmalloc/) 和 Microsoft 的 [mimalloc](https://microsoft.github.io/mimalloc)。

### 4.5. 测试/调试你的代码

#### 4.5.1. 做一个测试框架

AbstractMachine 代码的调试并不容易——不论是 native 还是运行在模拟器里，AM APIs 都和系统有紧密的耦合。大家有没有想过，你们的代码是否可以链接 `threads.h` 直接运行测试呢？答案当然是肯定的！在这个实验里，做一个测试框架对你找到 bug 其实是非常有用的。

你可以创建一个 test 目录，用于存放和测试相关的代码，例如我们提供的 `threads.h`:

```c
$ tree test 
test
├── am.h # 一个空的 am.h
├── common.h
├── test.c
└── threads.h # 上课时给出的代码
```

除此之外，所有的操作系统/框架代码等都和之前保持一致——我们通过最小的、非侵入式的项目修改确保你在任何时候都可以通过 `make submit` 直接提交代码而不需要做出任何额外的修改。这些自动工具/代码框架对于做实验 (包括做 Online Judge) 都是非常值得的。

为了让我们的代码能够兼容，你可以需要增加一些条件编译，例如，在 `pmm.c` 中，虽然你的 `kalloc` 和 `kfree` 的实现可以保持不变，但初始化代码则需要不同：

```c
#ifndef TEST
// 框架代码中的 pmm_init (在 AbstractMachine 中运行)
static void pmm_init() {
  uintptr_t pmsize = ((uintptr_t)heap.end - (uintptr_t)heap.start);
  printf("Got %d MiB heap: [%p, %p)\n", pmsize >> 20, heap.start, heap.end);
}
#else
// 测试代码的 pmm_init ()
static void pmm_init() {
  char *ptr  = malloc(HEAP_SIZE);
  heap.start = ptr;
  heap.end   = ptr + HEAP_SIZE;
  printf("Got %d MiB heap: [%p, %p)\n", HEAP_SIZE >> 20, heap.start, heap.end);
}
#endif
```

在你相应完成两个不同环境的条件编译后，在你的 Makefile 里增加一个编译目标 (我们添加了 git 的依赖，使得你的提交被正确追踪)：

```c
test: git
        @gcc $(shell find src/ -name "*.c")  \
             $(shell find test/ -name "*.c") \
             -Iframework -Itest -DTEST -lpthread \
             -o build/test
        @build/test
```

然后就可以愉快地写测试代码啦 (`test.c`):

```c
static void entry(int tid) { pmm->alloc(128); }
static void goodbye()      { printf("End.\n"); }
int main() {
  pmm->init();
  for (int i = 0; i < 4; i++)
    create(entry);
  join(goodbye);
}
```

> #### 把时间投资在框架建设上
>
> 你可能需要花一点时间才能完成 `make` 和 `make test` 共用同一份 kalloc/kfree 代码的框架。但这绝对是值得的。一旦完成，直接测试/调试本地代码的好处是巨大的——它节省了编译、运行、调试……整个开发流程的 overhead。任何想用 “蛮力” 把实验糊弄过去都将导致巨大的时间浪费。

### 4.5.2. 设计测试用例

如果不想被 Online Judge 折磨得死去活来，你应当设计一个自己的测试框架，尝试不同类型的测试，例如：

- 针对数据结构的单元测试；
- 最基本的 smoke test，测试单线程版本是否正确；
- 基本的并发测试，例如多个处理器并发地分配 (但不回收)；
- 针对回收专门设计的测试；
- 极端情况下的压力测试 (我们会这么做！)
  - 在多个处理器上模拟各种不同类型的内存访问模式
    - 频繁的小内存申请；
    - 频繁的大内存申请；
    - 混合的内存申请；
    - 多处理器竞争的申请/释放……
  - 你还需要有个办法验证你的 kalloc/free 是否正确
    - 一个可行的方案是，每次分配/释放都用 printf 打印一条记录，然后再写一个小脚本解析记录，确保例如同一段内存没有被分配给两个 kalloc。

这时候，代码框架的威力就显示出来了！你可以很容易地批量运行很多测试，例如

```
int main(int argc, char *argv[]) {
  if (argc < 2) exit(1);
  switch(atoi(argv[1])) {
    case 0: do_test_0();
    case 1: do_test_1();
    ...
  }
}
```

然后在你的 Makefile 里批量运行它们：

```
testall: test
        @build/test 0
        @build/test 1
        @build/test 2
        ...
```

> #### 警惕并发
>
> 很可能你的顺序测试完全通过，但多处理器跑起来就挂了。不要慌，多加点 printf logs，慢慢诊断可能的问题。然后你可能会惊奇地发现，也许加了一个 printf，错误就不见了。此时的建议是调整 workloads、增加延迟等保证 bug 的稳定再现，然后再进行诊断。如果你希望实现各种 fancy 的算法，不妨从一把大锁保护 kalloc/free 开始。这是个不错的主意——这样你一开始就只需要关注单线程 kalloc/free 的正确性。当你需要性能的时候，再逐步把锁拆开。

### 4.5.3 性能调优

> #### 抛开 workload 谈优化，就是耍流氓
>
> *Premature optimization is the root of all evil.* —— D. E. Knuth
>
> 要考虑好你优化的场景，否则一切都是空谈。

你需要选取适当的 workload 进行调优，并且理解你程序的性能瓶颈在哪里。这时候，靠 “猜” 和随机修改程序以获得性能提升的方式是不专业的表现。理解程序性能的最好方法是使用正确的工具：profiler。作为本地进程运行的测试用例也能更容易地 profile——你可以用 Linux 系统自带的各种工具找到你实现的性能瓶颈。当然，这个实验对性能的要求并不高，大家体验一下即可。

当然，这个实验给大家的剧透是：如果数据结构选择得不太差，性能瓶颈就主要在于多个线程并发访问时的互斥，由于多个线程都需要获得同一把锁，就会出现 “一核出力、他人围观” 的情况。正如课本上所说，现代 malloc/free 实现的主要思想就是区分 fast/slow path，使得 fast path 分配在线程本地完成，从而不会造成锁的争抢。你也可以为每个 CPU 分配页面的缓存——可以借鉴 slab，也可以预先把页面分配给 CPU，在内存不足时再上锁从其他 CPU “偷取” 页面。

## 5. 操作系统中的内存管理

很多现代编程语言都运行在自动管理内存的的运行时环境上：Java, Python, Javascript, ...它们的特点是：没有 raw pointer，也没有 API 用于 “释放” 内存。当一个对象从一系列的 root objects (例如栈上的局部变量、全局变量等……) 通过引用关系不可达，它们就会自动被运行时环境回收。这真是程序员的福音——再也没有 use-after-free, double-free 等问题了，memory leak 的可能性也大大降低 (虽然依然可能会有)。

当然，自动内存管理带来的问题是内存回收是个 highly non-trivial 的问题，至今仍然不能算彻底完美地解决，生产系统依然经常受到垃圾回收 (回收不可达对象) 停顿/降低性能的困扰。由于操作系统的性能至关重要，让垃圾回收占据处理器运行似乎不是个好主意……未必！

- 阅读材料：Cody Cutler, M. Frans Kaashoek, and Robert T. Morris. [The benefits and costs of writing a POSIX kernel in a high-level language](https://www.usenix.org/conference/osdi18/presentation/cutler). In *Proc. of OSDI*, 2018. (硬核/专业人士警告)

当然，操作系统内核的有趣故事也没有停过：

- Kevin Boos, Namitha Liyanage, Ramla Ijaz, and Lin Zhong. [Theseus: An experiment in operating system structure and state management](https://www.usenix.org/conference/osdi20/presentation/boos). In *Proc. of OSDI*, 2020.