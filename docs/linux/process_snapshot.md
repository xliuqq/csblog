## 进程检查点和恢复

## BCLR

> [Berkeley Lab Checkpoint/Restart](https://crd.lbl.gov/divisions/amcr/computer-science-amcr/class/research/past-projects/BLCR/)

用户态的libcr库和kernel module来完成相关的Checkpoint/Restart工作。**最新版本0.8.5，2013年。**



## CRIU

> [CheckPoint/Restart in user space](https://criu.org/Main_Page)，在用户态实现进程的Checkpoint/Restart。

**支持目前云计算领域最火的docker容器的c/r**；

冻结一个正在运行的程序，并且checkpoint它到一系列的文件，然后你就可以使用这些文件在任何主机重新恢复这个程序到被冻结的那个点(白话就是**实现对已运行程序的备份和恢复**)。所以criu通常被用在程序或者容器的热迁移、快照、远程调试等。

- 热迁移：程序的热迁移；
- 代码动态注入：执行一些代码去捕获当前进程相关信息；
- TCP sockets checkpoint-restore：有能力在不破坏一个TCP链接的情况下，备份和恢复它的状态。

