# 资源限制

### Resource

`setrlimit()`函数限制资源使用，设置特定资源上面的软限制和硬限制。

软限制是一个值，当超过这个值的时候操作系统通常会发送一个信号来限制或通知该进程。 **硬限制是用来指定软限制能设定的最大值。通常来讲，这个由系统管理员通过设置系统级参数来决定**。

- **RLIMIT_AS**：maximum area (in bytes) of address space
- **RLIMIT_RSS**：maximum resident set size
- **RLIMIT_VMEM**：largest area of mapped memory



## Cgroups

> [Cgroups](https://github.com/francisbouvier/cgroups) ：A library to manage cgroups Linux kernel feature.

- 通过cgroups来限制cpu和memory的使用；
- 支持实时修改并生效；

### 安装

`pip install cgroups`

### 使用

```python
import os
import subprocess

from cgroups import Cgroup

# First we create the cgroup 'charlie' and we set it's cpu and memory limits
cg = Cgroup('charlie')
cg.set_cpu_limit(50)
cg.set_memory_limit(500)

# Then we a create a function to add a process in the cgroup
def in_my_cgroup():
	pid  = os.getpid()
	cg = Cgroup('charlie')
	cg.add(pid)

# And we pass this function to the preexec_fn parameter of the subprocess call
# in order to add the process to the cgroup
p1 = subprocess.Popen(['worker_1'], preexec_fn=in_my_cgroup)
p2 = subprocess.Popen(['worker_2'], preexec_fn=in_my_cgroup)
p3 = subprocess.Popen(['worker_3'], preexec_fn=in_my_cgroup)

# Processes worker_1, worker_2, and worker_3 are now in the cgroup 'charlie'
# and all limits of this cgroup apply to them

# We can change the cgroup limit while those process are still running
cg.set_cpu_limit(80)

# And of course we can add other applications to the cgroup
# Let's say we have an application running with pid 27033
cg.add(27033)
```

