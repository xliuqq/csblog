# 存储设备原理

**本讲内容**：

- 存储系统原理
- 持久数据的存储：磁、光、电

## 存储系统原理

### 为什么会有内存和外存？

状态机*(M,R)*

- Memory 内存
- Registers 寄存器 (原则上可以不需要)

存储 “当前状态” 的需求

- **可以寻址**：根据编号读写数据
- 访问速度尽可能快
  - 越快就意味着越难 persist
  - 允许状态<font color='red'>**在掉电后丢失**</font>

### “当前状态” 的存储方案

Delay line: 绳子

- 因为信号衰减，需要持续放大

**Magnetic core**: [磁铁](https://corememoryshield.com/index.html)

- “Segmentation fault (core dumped)”
- 什么是 core dump？
  - 一种 non-volatile memory

**SRAM/DRAM**: **Flip-Flop 和电容**

- 今天的实现方案

### 开始持久化之旅

> <font color='red'>**除了 “当前状态”，我们希望更大、更多的数据能 “留下来” **</font>(并且被操作系统有效地管理起来)

持久存储介质

- 构成一切文件的基础
  - 逻辑上是一个 `bit/byte array`
  - 根据局部性原理，允许我们按 **“大块” 读写**
- 评价方法：价格、容量、速度、可靠性
  - 再次见证人类文明的高光时刻！

## 持久化存储：磁

### 磁带 (Magnetic Tape, 1928)

1D 存储设备

- 把 Bits “卷起来”

  - 纸带上均匀粘上铁磁性颗粒

- 只需要一个机械部件 (转动) 定位

  - 读取：放大感应电流
  - 写入：电磁铁磁化磁针

  | <img src="pics/fuduji.jpg" alt="img" style="zoom: 50%;" /> | <img src="pics/rec.jpg" alt="img" style="zoom: 33%;" /> | <img src="pics/walkman.jpg" alt="img" style="zoom: 33%;" /> |
  | :--------------------------------------------------------: | ------------------------------------------------------- | ----------------------------------------------------------- |

### 磁带：作为存储设备的分析

分析：应用场景：<font color='red'>**冷数据的存档和备份**</font>

- 价格：**非常低** - 都是廉价的材料
- 容量：**非常高**
- 读写速度
  - 顺序读取：**勉强** - 需要等待定位
  - 随机读取：<font color='red'>**几乎完全不行**</font>
- 可靠性：<font color='red'>**存在机械部件、需要保存的环境苛刻**</font>

### 磁鼓 (Magnetic Drum, 1932)

1D → 1.5D (1D x n)

- 用**旋转的二维平面**存储数据
  - 无法内卷，容量变小
- 读写延迟不会超过**旋转周期**
  - 随机读写速度大幅提升

<img src="pics/mag-drum.jpg" alt="mag-drum" style="zoom: 20%;" />

### 磁盘 (Hard Disk, 1956)

1D → 2.5D (2D x n)

- 在二维平面上放置许多磁带 (内卷)

![img](pics/disk-mechanism.jpg)

克服许多工程挑战

![img](pics/hard-disk-mag.jpg)

### 磁盘：作为存储设备的分析

分析：应用场景：计算机系统的主力数据存储 (~~海量数据：便宜才是王道~~)

- 价格：**低** - 密度越高，成本越低
- 容量：**高 (2.5D)** - 平面上可以有数万个磁道
- 读写速度
  - 顺序读取：**较高**
  - 随机读取：**勉强**
- 可靠性：<font color='red'>**存在机械部件，磁头划伤盘片导致数据损坏**</font>

### 磁盘：性能调优

为了读/写一个扇区：存取时间 = 寻道时间 + 延迟时间 + 传输时间

1. 读写头需要到对应的磁道
   - 读写头移动时间通常也需要几个 ms
2. 转轴将盘片旋转到读写头的位置
   - 7200rpm → 120rps →  1/120 / 2 = 4.17 ms

通过缓存/调度等缓解

- 例如著名的 “电梯” 调度算法
- 现代 HDD 都有很好的 firmware 管理磁盘 I/O 调度
  - `/sys/block/[dev]/queue`
  - `[mq-deadline] none` (读优先；但写也不至于饿死)

### 软盘 (Floppy Disk, 1971)

把<font color='red'>**读写头和盘片分开**</font>——实现数据移动

- 计算机上的软盘驱动器 ([drive](https://www.bilibili.com/video/BV1BS4y1X76n)) + 可移动的盘片
  - 8" (1971), 5.25" (1975), 3.5" (1981)
    - 最初的软盘成本很低，就是个纸壳子
    - 3.5 英寸软盘为了提高可靠性，已经是 “硬” 的了

<img src="pics/floppy-disk.jpg" alt="floppy-disk" style="zoom: 80%;" />

### 软盘：作为存储设备的分析

分析：躺在博物馆供人参观，彻底被 USB Flash Disk 杀死

- 价格：**低** - 塑料、盘片和一些小材料
- 容量：**低** (暴露的存储介质，密度受限)
- 读写速度}：顺序/随机读取：**低**
- 可靠性：**低** (暴露的存储介质)

## 持久化存储：坑

> 天然容易 “阅读” 的数据存储：如同沙滩的SOS、纸带上的孔；

### Compact Disk (CD, 1980)

在反射平面 (1) 上挖上粗糙的坑 (0)

- 激光扫过表面，就能读出坑的信息来
  - 飞利浦 (碟片) 和索尼 (数字音频) 发明
  - ~700 MiB，在当时是非常巨大的容量

<img src="pics/cdplay.gif" alt="pics/cdplay.gif" style="zoom: 80%;" />

### CD-RW

能否克服只读的限制？

- 方法 1

  - 用激光器烧出一个坑来 (“刻盘”)

  - 使用**持久化数据结构** (append-only)
- 方法 2：改变材料的反光特性
  - PCM (Phase-change Material)
  - [How do rewriteable CDs work?](https://www.scientificamerican.com/article/how-do-rewriteable-cds-wo/)

### 挖坑的技术进展

> 在实用方面逐渐被网络淘汰

CD (740 MB)：780nm 红外激光

DVD (4.7 GB)：635nm 红色激光

Blue Ray (100 GB)：405nm 蓝紫色激光

### 光盘：作为存储设备的分析

分析：应用场景：作为数字收藏

- 价格：**很低** (而且很容易通过 “压盘” 复制)
- 容量：**高**
- 读写速度：
  - **顺序读取速度高；随机读取勉强**
  - <font color='red'>**写入速度低**</font> (挖坑容易填坑难)
- 可靠性：**高**

### “挖坑”：不止是数据存储

？挖坑的方式造芯片？

![EUV.png](pics/EUV.png)

## 持久化存储：电

### Solid State Drive (1991)

之前的持久存储介质都有致命的缺陷

- 磁：机械部件导致 ms 级延迟
- 坑 (光): 一旦挖坑，填坑很困难 (CD 是只读的)

密度和速度：最后还得靠<font color='red'>**电**</font> (电路) 解决问题

- Flash Memory “闪存”
  - Floating gate 的<font color='red'>**充电/放电**</font>实现 1-bit 信息的存储

![nand-flash](pics/nand-flash.jpg)

### Flash Memory: 几乎全是优点

分析

- 价格：低 (大规模集成电路，便宜)
- 容量：高 (3D 空间里每个 (𝑥,𝑦,𝑧) 都是一个 bit)
- 读写速度：高(直接通过电路读写)
  - 不讲道理的特性：容量越大，速度越快 (电路级并行)
  - 快到淘汰了旧的 SATA 接口标准 (NVMe)
- 可靠性：高 (没有机械部件，随便摔)

但有一个意想不到的缺点 (大家知道是什么吗？下章)

### USB Flash Disk (1999)

优盘容量大、速度快、相当便宜

- 很快就取代了软盘，成为了人手 *n* 个的存储介质
  - Compact Flash (CF, 1994)
  - USB Flash Disk (1999, “朗科”)

**放电 (erase) 做不到 100% 放干净**

- 放电<font color='red'>**数千/数万次**</font>以后，就好像是 “充电” 状态了
- Dead cell; “wear out”
  - 必须解决这个问题 SSD 才能实用

![pics/upan.jpg](pics/upan.jpg)

### 优盘和 SSD 的区别

优盘, SD 卡, SSD 都是 **NAND(Not AND) Flash**。

但软件/硬件系统的复杂程度不同，效率/寿命也不同

- 典型的 SSD
  - CPU, on-chip RAM, 缓存, store buffer, 操作系统 ...
  - 寿命: ~1 PiB 数据写入 (~1,000 年寿命)

- SD 卡：<font color='red'>**一定不要用便宜的优盘保存重要数据**</font>
  - SDHC 标准未规定
    - 黑心商家一定会偷工减料
  - 但良心厂家依然有 [ARM 芯片](https://www.bunniestudios.com/blog/?p=898)

### SSD 的细节

软件定义磁盘：**每一个 SSD 里都藏了一个完整的计算机系统**

![m2ssd](pics/m2ssd.png)

- FTL: Flash Translation Layer
  - “Wear Leveling”: 软件管理那些可能出问题的 blocks
  - 像是 managed runtime (with garbage collection)

**Wear Leveling**：Logical block address (LBA) → Physical block address (PBA)

- SSD 的 Page/Block 两层结构
  - Page (读取的最小单位, e.g., 4KB)
  - Block (写入的最小单位, e.g., 4MB)
  - Read/write amplification (读/写不必要多的内容)
- Copy on write
- [Coding for SSDs](https://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)

### FTL 带来的性能、可靠性、安全性问题

理解了 FTL 之后

- <font color='red'>**即便格式化后写入数据 (不写满)**</font>
  - 同一个 logic block 被覆盖，**physical block 依然存储了数据 (copy-on-write)**
  - 需要文件系统加密

另一个 memory system 相关的安全问题

- [Row Hammer(TCAD'19)](https://arxiv.org/pdf/1904.09724)
  - 更重的负载可能会 “干扰” 临近的 DRAM Cells

硬件里的软件：其实非常复杂：算法, cache; store buffer; ...

- 造疯狂的 workloads，把它弄挂吧！
  - [Understanding the robustness of SSDs under power fault](https://www.usenix.org/system/files/conference/fast13/fast13-final80.pdf) (FAST'13)

## 阅读材料

如果更快的 non-volatile memory 到来，我们的计算机系统是否会发生翻天覆地的变化？

- [Operating system implications of fast, cheap, non-volatile memory](https://dl.acm.org/doi/10.5555/1991596.1991599) (HotOS'13)

欢迎到大家阅读课堂中的一些 blogs，以及自己收集一些有趣的资料，例如 [How do rewriteable CDs work?](https://www.scientificamerican.com/article/how-do-rewriteable-cds-wo/) 和 [Coding for SSDs](https://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)。

### 1. 阅读材料

教科书 Operating Systems: Three Easy Pieces:

- 第 37 章 - [Hard Disk Drives](./book_os_three_pieces/37-file-disks.pdf)
- 第 44 章 - [Flash-based SSDs](./book_os_three_pieces/44-file-ssd.pdf)
