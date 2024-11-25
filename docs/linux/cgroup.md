# CGroup

> 在当前内核版本（3.x）下，开启了kmem accounting功能，会导致 memory cgroup 的条目泄漏无法回收（无法正常计数）。
>
> - 内核日志的“SLUB: Unable to allocate memory on node -1”报错。

cgroup，其本身的作用只是任务跟踪。但其它系统（比如cpusets，cpuacct），可以利用cgroup的这个功能实现一些新的属性，比如统计或者控制一个cgroup中进程可以访问的资源。举个例子，cpusets子系统可以将进程绑定到特定的cpu和内存节点上。

**control groups 理解为 controller （system resource）（for）（process）groups**，以一组进程为目标进行系统资源分配和控制。



## 功能

- 资源限制：任务的资源总额进行限制；
- 优先级分配：通过分配的CPU时间片数量及磁盘IO带宽大小，相当于控制任务优先级；
- 资源统计：统计系统的资源使用量，如CPU使用时长、内存用量等，适用于计费；
- 任务控制：对任务执行进行**挂起、恢复**等操作；



## 版本(v1/v2)

Linux 中有两个 cgroup 版本：cgroup v1 和 cgroup v2。cgroup v2 是 Linux `cgroup` API 的下一个版本，v2 不兼容 v1。

cgroup v2 提供一个具有增强资源管理能力的统一控制系统。 cgroup v2 对 cgroup v1 进行多项改进：

- API 中单个统一的层次结构设计，**v2 只有单个层级树**
- 更安全的子树委派给容器
- 更新的功能特性， 例如[压力阻塞信息（Pressure Stall Information，PSI）](https://www.kernel.org/doc/html/latest/accounting/psi.html)
- **跨多个资源**的增强资源分配管理和隔离
- 统一核算不同类型的内存分配（网络内存、内核内存等）
- 考虑非即时资源变化，例如页面缓存回写

一些 Kubernetes 特性专门使用 cgroup v2 来增强资源管理和隔离。

- Ubuntu（从 21.10 开始，推荐 22.04+）

```shell
$ mount | grep cgroup2
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime)
```

判断是否使用 cgroup v2：Yes if `/sys/fs/cgroup/cgroup.controllers` is present.



## 基本概念

### Task

系统进程或线程

### cgroup 

控制组，它提供了一套机制用于**控制<font color='red'>一组特定进程</font>对资源的使用**。

**关联一组task和一组subsystem的配置参数**。一个task对应一个进程，cgroup是资源分片的最小单位。

### subsystem

子系统，一个通过cgroup提供的工具和接口来管理进程集合的模块。一个子系统就是一个典型的“资源控制器”，用来**调度资源或者控制资源使用的上限**。其实**每种资源就是一个子系统**。子系统可以是以进程为单位的任何东西，比如虚拟化子系统、内存子系统。

### hierarchy

层级树，多个cgroup的集合，这些集合构成的树叫hierarchy。

可以认为这是一个资源树，**附着在这上面的进程可以使用的资源上限必须受树上节点（cgroup）的控制**。hierarchy上的层次关系通过cgroupfs虚拟文件系统显示。

系统允许多个hierarchy同时存在，每个hierachy包含系统中的部分或者全部进程集合。

### cgroupfs

用户管理操纵cgroup的主要接口：

- 通过在cgroupfs文件系统中创建目录，实现cgroup的创建；
- 通过向目录下的属性文件写入内容，设置cgroup对资源的控制；
- 向task属性文件写入进程ID，可以将进程绑定到某个cgroup，以此达到控制进程资源使用的目的；
- 列出cgroup包含的进程pid。



## subsystem配置参数介绍

### blkio - BLOCK IO 资源控制

> 控制子系统可以限制进程读写的 **IOPS 和吞吐量**，但它只能对 Direct I/O 的文件读写进行限速，对 Buffered I/O 的文件读写无法限制。
>
> Buffered I/O 指会**经过 PageCache** 然后再写入到存储设备中。这里面的 Buffered 的含义跟内存中 buffer cache 不同，这里的 Buffered 含义相当于内存中的buffer cache+page cache。
>
> 在 Linux 中，**文件默认读写方式为 Buffered I/O**，应用程序一般将文件写入到 PageCache 后就直接返回了，然后**内核线程会异步**将数据从内存同步到磁盘中。
>
> cgroup v2 使用了统一层级（unified hierarchy)，各个子系统都可以挂载在统一层级下，一个进程属于一个控制组，每个控制组里可以定义自己需要的多个子系统。cgroup v2 中 io 子系统等同于 v1 中的 blkio 子系统。

限额类是主要有两种策略：

- 一种是基于完全公平队列调度（CFQ：Completely Fair Queuing ）的按权重分配各个 cgroup 所能占用总体资源的百分比，好处是当资源空闲时可以充分利用，但只能用于最底层节点 cgroup 的配置；

- 另一种则是设定资源使用上限，这种限额在各个层次的 cgroup 都可以配置，但这种限制较为生硬，并且容器之间依然会出现资源的竞争。

### cpu - CPU 资源控制

CPU 资源的控制也有两种策略：

- 一种是完全公平调度 （CFS：Completely Fair Scheduler）策略，提供了限额和按比例分配两种方式进行资源控制；
  - `cpu.cfs_period_us`：cpu分配的周期(微秒），默认为100000
  - `cpu.cfs_quota_us`：该control group限制占用的时间（微秒），默认为-1，表示不限制。如果设为50000，表示占用50000/100000=50%的CPU。
  - `cpu.shares`：***对于控制组之间的 CPU 分配比例***，它的缺省值是 1024，当所有的控制组的CPU总额超过节点CPU时起作用；
    - ***无论其他容器申请多少 CPU 资源，即使运行时整个节点的 CPU 都被占满的情况下，我的这个容器还是可以保证获得需要的 CPU 数目***

- 另一种是实时调度（RT：Real-Time Scheduler）策略，针对实时进程按周期分配固定的运行时间。配置时间都以微秒（µs）为单位，文件名中用us表示

### cpuacct - CPU 资源报告

### cpuset - CPU 绑定

### device - 限制 task 对 device 的使用

### freezer - 暂停 / 恢复 cgroup 中的 task

### memory - 内存资源管理

- `memory.limit_in_bytes`：物理内存使用限制
- `memory.soft_limit_in_bytes`：限制的内存软额度
- `memory.memsw.limit_in_bytes`：限制swap+memory的大小

- `memory.oom_disable`的值来设置内存超出设定值时是操作系统kill进程还是休眠进程。
- `memory.swappiness` 文件的值修改为 0，进程不使用Swap分区（重启失效）。

### net_prio - 网络设备优先级

### net_cls - 配合tc进行网络限制

### perf_event - 统一的性能测试

cgroup中的任务可以进行统一的性能测试。 

### pid - 限制总任务数

> linux kernel >= 4.3

限制cgroup及其所有子孙cgroup里面能创建的总的task数量。

`pids.current`: 表示当前cgroup及其所有子孙cgroup中现有的总的进程数量

`pids.max`: 当前cgroup及其所有子孙cgroup中所允许创建的总的最大进程数量

## cgroup操作准则与方法

### 准则

#### 一个层级可以有多个子系统

mount 的时候hierarchy可以attach多个subsystem，如CPU和Memory的子系统附加到同一个层级。



#### 一个已经被挂载的子系统只能被再次挂载在一个空的层级上

已经mount一个subsystem的hierarchy不能挂载一个已经被其它hierarchy挂载的subsystem



#### 每个task只能在同一个hierarchy的唯一一个cgroup里

不能在同一个hierarchy下有超过一个cgroup的tasks里同时有这个进程的pid



#### 子进程在被fork出时自动继承父进程所在cgroup，但是fork之后就可以按需调整到其他cgroup



### 方法

#### 层级（Hierarchy ）

```shell
# 创建了一个Hierarchy（/sys/fs/cgroup 只读？）
$ mkdir /sys/fs/cgroup/cpu_and_mem 
# 挂载生效，并附加了cpu和cpuset两个子系统。
$ mount -t cgroup -o cpu,cpuset cpu_and_mem /sys/fs/cgroup/cpu_and_mem/ 
# 附加memory子系统
$ mount -t cgroup -o remount,cpu,cpuset,memory cpu_and_mem /sys/fs/cgroup/cpu_and_mem/
# 剥离memory子系统
$ mount -t cgroup -o remount,cpu cpu_and_mem /sys/fs/cgroup/cpu_and_mem/
# 删除层级
$ umount /sys/fs/cgroup/cpu_and_mem/
```

#### Control Group

创建Hierarchy的时候其实已经创建了一个Control Group叫做root group，其表现形式就是一个目录，一个目录就是一个Control Group。

```shell
# -t指定的用户或者组对tasks文件的拥有权
# -a指定的用户或者组对其它文件的拥有权
# -g 指定要附加的子系统（必须是层级拥有的）和Control Groups路径
$ cgcreate -t uid:gid -a uid:gid -g subsystems:path

# cgcreate会根据-g后面指定的子系统，找到包含这些子系统的所有Hierarchy，并为其创建control group
$ cgcreate -g cpuset:/subgroup

# 删除的时候也要指定子系统，会根据子系统找到所有的Hierarchy 中的指定Control Group来删除
$ cgdelete cpuset,memory:/subgroup
```

## seccomp

linuxkernel从2.6.23版本开始所支持的一种安全机制。

在Linux系统里，大量的系统调用（systemcall）直接暴露给用户态程序。但是，并不是所有的系统调用都被需要，而且不安全的代码滥用系统调用会对系统造成安全威胁。**通过seccomp限制程序使用某些系统调用**，这样可以减少系统的暴露面，同时是程序进入一种“安全”的状态。

1. *从*Linux3.8开始，可以利用`/proc/[pid]/status`中的Seccomp字段查看。如果没有seccomp字段，说明内核不支持seccomp*。*



## 驱动

**systemd和cgroupfs都是CGroup管理器，而systemd是大多数Linux发行版原生的。**

- **同时使用cgroupfs和systemd的节点，在资源紧张时会变得不稳定。**

当选择systemd作为Linux发行版的init system时，init proccess生成并使用root控制组（/sys/fs/cgroup），并充当CGroup管理器。



## 使用示例

### Linux是否启用cgroup

```shell
$ uname -r
# 4.18.0-24-generic
$ cat /boot/config-4.18.0-24-generic | grep CGROUP
# 对应的CGROUP项为“y”代表已经打开linux cgroups功能。
```

Centos默认已经为**每个子系统创建单独Hierarchy** ，进行查看：

```shell
$ mount -t cgroup
tmpfs /sys/fs/cgroup tmpfs ro,seclabel,nosuid,nodev,noexec,mode=755 0 0
cgroup /sys/fs/cgroup/systemd cgroup rw,seclabel,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd 0 0
cgroup /sys/fs/cgroup/hugetlb cgroup rw,seclabel,nosuid,nodev,noexec,relatime,hugetlb 0 0
cgroup /sys/fs/cgroup/cpu,cpuacct cgroup rw,seclabel,nosuid,nodev,noexec,relatime,cpuacct,cpu 0 0
cgroup /sys/fs/cgroup/perf_event cgroup rw,seclabel,nosuid,nodev,noexec,relatime,perf_event 0 0
cgroup /sys/fs/cgroup/net_cls,net_prio cgroup rw,seclabel,nosuid,nodev,noexec,relatime,net_prio,net_cls 0 0
cgroup /sys/fs/cgroup/cpuset cgroup rw,seclabel,nosuid,nodev,noexec,relatime,cpuset 0 0
cgroup /sys/fs/cgroup/memory cgroup rw,seclabel,nosuid,nodev,noexec,relatime,memory 0 0
cgroup /sys/fs/cgroup/blkio cgroup rw,seclabel,nosuid,nodev,noexec,relatime,blkio 0 0
cgroup /sys/fs/cgroup/devices cgroup rw,seclabel,nosuid,nodev,noexec,relatime,devices 0 0
cgroup /sys/fs/cgroup/pids cgroup rw,seclabel,nosuid,nodev,noexec,relatime,pids 0 0
cgroup /sys/fs/cgroup/freezer cgroup rw,seclabel,nosuid,nodev,noexec,relatime,freezer 0 0
```

### 显示CPU使用比例

在目录`/sys/fs/cgroup/cpu`下创建文件夹，就创建了一个control group

root 用户，写一个死循环的代码，将CPU使用率通过cgroup限制在 20%

```shell
# 在/sys/fs/cgroup/cpu/ 创建控制组/层级
$ mkdir /sys/fs/cgroup/cpu/test
# 创建成功后，自动创建cpu配置文件
$ cd /sys/fs/cgroup/cpu/test
$ ls
# cgroup.clone_children  cgroup.procs  cpuacct.stat  cpuacct.usage  cpuacct.usage_percpu  # cpu.cfs_period_us  cpu.cfs_quota_us  cpu.shares  cpu.stat  notify_on_release  tasks

$ cat cpu.cfs_quota_us
# -1
# 如果cgroup只读，执行 mount -o remount,rw /sys/fs/cgroup
# 限制进程只占用 20%的CPU
$ echo 100000 > cpu.cfs_period_us
$ echo 20000 > cpu.cfs_quota_us
# 将进程号写入tasks里
$ echo 30167 > tasks

# 删除cgroup (要么停止进程，要么将进程从tasks文件里删除)
rmdir tasks
```

### 限制进程可创建的子进程数

```shell
$ mount | grep pids
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
$ cd /sys/fs/cgroup/pids/
$ mkdir test
$ cd test/
$ ls -l
total 0
-rw-r--r-- 1 root root 0 Nov 23 16:18 cgroup.clone_children
--w--w--w- 1 root root 0 Nov 23 16:18 cgroup.event_control
-rw-r--r-- 1 root root 0 Nov 23 16:18 cgroup.procs
-rw-r--r-- 1 root root 0 Nov 23 16:18 notify_on_release
-r--r--r-- 1 root root 0 Nov 23 16:18 pids.current
-rw-r--r-- 1 root root 0 Nov 23 16:18 pids.max
-rw-r--r-- 1 root root 0 Nov 23 16:18 tasks
# 将pids.max设置为1，即当前cgroup只允许有一个进程
$ echo 1 >pids.max 
# 将当前bash进程加入到该cgroup
$ echo $$ > cgroup.procs
$ /bin/echo "Here's some processes for you." | cat
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
```

### 如何支持GPU

> Linux中每个设备都有一个设备号，设备号由主设备号和次设备号两部分组成。

```shell
#显示GPU的主设备号和次设备号
$ ls -l /dev | grep nvidia 
crw-rw-rw-   1 root root    195,   0 Nov  9 13:34 nvidia0
crw-rw-rw-   1 root root    195,   1 Nov  9 13:34 nvidia1
crw-rw-rw-   1 root root    195,   2 Nov  9 13:34 nvidia2
crw-rw-rw-   1 root root    195,   3 Nov  9 13:34 nvidia3

#只显示主设备号
$ cat /proc/devices
```

 cgroup 禁用设备的访问

```shell
# the c indicates that /dev/tty is a character device, 195,0 is major and minor numbers of the device. The last w is write permission, so the above command forbids tasks to write to the /dev/tty.
$ echo "c 195:0 w" > devices.deny

```

创建进程死循环`test.sh`执行 nvidia-smi，记录 test.sh 的进程号

```shell
# test.sh
while true; do
	nvidia-smi
	sleep 1
done
```

将进程号写到 tasks 里面，可以发现上面的脚本显示的GPU少了一个

```txt
Thu May 18 19:11:30 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 460.73.01    Driver Version: 460.73.01    CUDA Version: 11.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:3B:00.0 Off |                    0 |
| N/A   28C    P8    10W /  70W |      0MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
No devices were found
No devices were found
No devices were found
```



### 使用`group`限制单用户/多用户计算资源

> yum install libcgroup-tools
>
> apt install cgroup-tools

在 `/etc/cgconfig.conf`添加如下配置，将会对组users_mem_limit 内的用户所能申请的最大内存进行限制。

```conf
# template 表示模板，对于每个用户单独限制
template users/%u {
    memory {
        memory.limit_in_bytes = 200M;
        memory.memsw.limit_in_bytes = 0M;
    }   
    cpu {
        cpu.cfs_quota_us = 200000;
        cpu.cfs_period_us = 100000;
    }   
}
# group是特殊字段，`rootg`是组的名称
# 属于同一组的是共享资源限制
group rootg {
    memory {
        memory.limit_in_bytes = 500M;
    }   
    cpu {
        cpu.cfs_quota_us = 400000;
        cpu.cfs_period_us = 100000;
    }   
}
```

`/etc/cgrules.conf`添加如下配置，实现将某个/某些用户添加到该组。

```conf
#用户名			   #限制类型        		#所属组
root    			cpu,memory              rootg
*       			cpu,memory              users/%u
```

设置`cgconfig.service`开启自启动

- 当服务启动以后，在`/sys/fs/cgroup/cpu/`和`/sys/fs/cgroup/memory/`目录下，会出现以 `root`命名和`users`命名的目录。
- `root`目录下，即是对root用户的资源限制，`users`目录下是登陆到此节点上普通用户的资源限制。

```shell
#开机启动
systemctl enable cgconfig
systemctl enable cgred
#重启服务
systemctl restart cgconfig
systemctl restart cgred
```

