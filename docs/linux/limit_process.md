# 进程的限制

**kernel.pid_max** ：系统的最大进程数
```shell
# 查看kernel.pid_max
sysctl kernel.pid_max
# 修改kernel.pid_max
sysctl -w kernel.pid_max=<new_value>
```

**kernel.threads-max** ：系统的最大线程数

```shell
# 查看kernel.threads-max
sysctl kernel.threads-max
# 修改kernel.threads-max
sysctl -w kernel.threads-max=<new_value>
```

**ulimit -u** : 用户的最大进程数（和线程数有关系么？）

```shell
# root默认是即系统线程数（kernel.threads-max）的一半
# 普通账号默认是/etc/security/limits.d/20-nproc.conf文件中配置
```

**ulimit -n** : 用户打开的最大文件限制

**修改/etc/security/limits.conf文件，root用户**

```
* soft nofile 1048576
* hard nofile 1048576
```

**vm.max_map_count**：**限制一个进程可以拥有的VMA(虚拟内存区域)的数量**。虚拟内存区域是一个连续的虚拟地址空间区域。在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。**调优这个值将限制进程可拥有VMA的数量**。限制一个进程拥有VMA的总数可能导致应用程序出错，因为当进程达到了VMA上线但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误。如果你的操作系统在NORMAL区域仅占用少量的内存，那么调低这个值可以帮助释放内存给内核用。

在其他资源可用的前提下，**单个JVM能开启的最大线程数是/proc/sys/vm/max_map_count的设置数的一半**。

```shell
sysctl -w vm.max_map_count=262144
```



