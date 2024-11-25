# 文件系统

## Bootfs

> 包含 bootloader和 Kernel，bootloader主要是引导加 kernel，Linux刚启动时会加bootfs文件系统，在 Docker镜像的最底层是bootfs.

当bootfs加载完成之后**整个内核就都在内存中**，此时内存的使用权已由 bootfs转交给内核，此时系统也会卸载bootfs。



## Rootfs

> 根文件系统是**内核启动时所mount的第一个文件系统**，内核代码映像文件保存在根文件系统中，而系统引导启动程序会在根文件系统挂载之后从中把一些基本的初始化脚本和服务等加载到内存中去运行。

包含的就是典型 Linux系统中的/dev、/proc、/bin、/etc等标准目录和文件，rootfs就是各种不同的操作系统发行版，比如：Ubuntu,、CentOS。



## RamDisk

虚拟内存盘是通过软件将一部分内存（RAM）模拟为硬盘来使用的一种技术，Alluxio软件就是将 RAMDisk 作为其存储介质。

分为`Ramdisk`, `ramfs`, `tmpfs`：

- Ramdisk 是传统意义上磁盘，可以格式化，然后加载；

  - Linux 2.0/2.2支持，大小固定，不能改变；
  - 内核编译时将Block device中Ramdisk支持选上，

- `ramfs` 和 `tmpfs` 是Linux内核2.4支持，大小可以自动增加或减少；

  - `ramfs` 内存文件系统，它处于虚拟文件系统(VFS)层，无需格式化，**默认一半内存大小**
  - 可以在 mount 时限定大小，不会自动增长
  - **ramfs不会用swap**，**tmpfs会使用swap**

  

### Linux下的挂载

``` shell
mkdir -p /media/nameme

mount -t tmpfs -o size=2048M tmpfs /mnt/ramdisk

mount -t ramfs -o size=2048M ramfs /mnt/ramdisk
```



### 性能测试

```shell
# 测试磁盘的写速率
$ time dd if=/dev/zero bs=4096 count=2000000 of=./8Gb.file
2000000+0 records in
2000000+0 records out
8192000000 bytes (8.2 GB) copied, 6.37105 s, 1.3 GB/s

real    0m6.374s
user    0m0.909s
sys     0m5.450s

# 测试磁盘读
$ time dd if=/dev/mapper/VolGroup00-LogVol00 of=/dev/null bs=8k
 2498560+0 records in
 2498560+0 records out
 247.99s real 1.92s user 48.64s system
```



## Proc文件系统

https://man7.org/linux/man-pages/man5/proc.5.html

虚拟内存：是进程运行时全部内存空间的总和，Java的-Xmx分配的是虚拟内存的空间；

### /proc/self

当前进程的信息，不同线程访问时获取的其线程的信息。

### /proc/[pid]/io

**rchar**：从存储器读取的字节数，是read(2)类似系统调用的字节数之和，**跟实际物理磁盘读取无关**；

**wchar**：写入磁盘的字节数，类似rchar；

**syscr**：读的系统调用

**syscw**：写的系统调用

**read_bytes**：**进程实际读取从存储器中的字节数**

**write_bytes**：进程实际写入从存储器中的字节数

### /proc/sat

```shell
# cpu   user     nice   system   idle     iowait irq softirq steal guest   guest_nice 
 cpu    10132153 290696 3084719 46828483  16683   0   25195   0     175628   0
 cpu0   1393280  32966  572056  13343292  6130    0   17875   0     23933    0
```

时间的单位是`USER_HZ`，

**user**：用户模式下花费的时间；

**nice**：用户模式下以低优先级花费的时间；

**system**：系统模式下花费的时间；

**idle**：空闲的时间

**iowait**：等待I/O的时间，由于以下原因，**不是很可靠**

- 任务等待I/O，而不是CPU等待，CPU空闲时会有其他任务被调度在这个CPU上；
- 在多核CPU上，等待I/O的任务未在任何CPU上运行，因此每个CPU的**iowait**是很难计算的；
- 该字段的值可能会降低

### /proc/[pid]/stat

进程的状态信息

### /proc/[pid]/status

`/proc/[pid]/stat` 和`/proc/[pid]/statm`的大部分信息，可阅读性好。

**VmPeak**：虚拟内存的峰值；

**VmSize**：虚拟内存大小；

**VmRss**：应用程序正在**使用的物理内存的大小**，**不精确**，ps命令的参数rss的值 (rss)；

**VmHWM**：应用程序正在**使用的物理内存的峰值**，**不精确**；

**VmLck**：锁定的物理内存大小，锁住的物理内存不能交换到硬盘 (locked_vm)；

**VmPin**：Pinned的物理内存大小，不能移动；

**VmData**：程序数据段的大小（所占虚拟内存的大小）
