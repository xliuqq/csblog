# CGroup

## 容器（Container）

yarn container 默认不支持对cpu进行资源隔离，例如申请的1个vcore，实际上又启动了多线程，还有GC线程等都会造成资源使用不可控。

内存限制，通过 Java 启动指定-Xmx，也不是Container真实的内存限制。

### 类型

#### DefaultContainerExecutor

- 默认的executor，管理容器的执行；
- 容器进程的用户是NodeManager的Unix User；
- 安全性低且没有任何CPU资源隔离机制。

#### LinuxContainerExecutor

容器进程的用户：

- 开启 full security 时，是提交应用的 Yarn User（因此需要在对应的NodeManager中，**该用户存在**，多租户下可能会有问题）；
- 未开启 full security时，默认为 nobody用户，通过配置可以修改：
  - `yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users`：是否在非安全模式下，限制运行用户为单个用户；
  - `yarn.nodemanager.linux-container-executor.nonsecure-mode.local-user`：非安全模式下的运行用户；

| yarn.nodemanager.linux-container-executor.nonsecure-mode.local-user | yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users | User running jobs         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :------------------------ |
| (default：nobody)                                            | (default：true)                                              | nobody                    |
| yarn                                                         | (default：true)                                              | yarn                      |
| yarn                                                         | false                                                        | (User submitting the job) |

## Cgroup 配置

> 前提：cgroup已经安装好，在 `/sys/fs/cgroup` 目录下。

**yarn-site.xml** （cgroup专属配置）

```xml
<property> <!-- 使用cgroup 必须是 LinuxContainerExecutor -->
  <name>yarn.nodemanager.container-executor.class</name>
  <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
</property>
<property> <!-- 启动cgroup 资源隔离 -->
  <name>yarn.nodemanager.linux-container-executor.resources-handler.class</name>
  <value>org.apache.hadoop.yarn.server.nodemanager.util.CgroupsLCEResourcesHandler</value>
</property>
<property> <!-- yarn使用的cgroup组,默认为/hadoop-yarn -->
    <name>yarn.nodemanager.linux-container-executor.cgroups.hierarchy</name>
    <value>/hadoop-yarn</value>
</property>
<property> <!-- 是否自动挂载cgroup, centos 7无需 -->
    <name>yarn.nodemanager.linux-container-executor.cgroups.mount</name>
    <value>false</value>
</property>
<property> <!-- cgroup挂载目录, centos 7之后无需，自动发现 -->
    <name>yarn.nodemanager.linux-container-executor.cgroups.mount-path</name>
    <value>/sys/fs/cgroup</value>
</property>
<property> <!-- NodeManager的Unix用户组，需要和container-executor.cfg中配置一样 -->
    <name>yarn.nodemanager.linux-container-executor.group</name>
    <value>hadoop</value>
</property>

<property> <!-- NodeManager 可以使用的cpu利用率的上限，默认100%  -->
	<name>yarn.nodemanager.resource.percentage-physical-cpu-limit</name>
    <value>100</value>
</property>
<property> <!-- container是否严格限制cpu份额（即使有空闲的cpu），默认false  -->
	<name>yarn.nodemanager.linux-container-executor.cgroups.strict-resource-usage</name>
    <value>false</value>
</property>
```

## Cgroup和安全

默认非安全模式，LCE 以用户 ”nobody“ 启动，可以配置为统一的默认用户`local-user`或者提交应用的用户。

| yarn.nodemanager.linux-container-executor.nonsecure-mode.local-user | yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users | User running jobs         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :------------------------ |
| (default：nobody)                                            | (default：true)                                              | nobody                    |
| yarn                                                         | (default：true)                                              | yarn                      |
| yarn                                                         | false                                                        | (User submitting the job) |

## Yarn Secure Container

> Secure Containers work only in the context of **secured YARN clusters**
>
> secured YARN cluster 是什么？这句话怎么理解？

Container 必须以提交应用的用户执行。

用户和用户组配置示例：**NodeManager以 yarn 用户启动，yarn所属组为 users 和 hadoop，users组包含yarn和其它用户（应用提交者，如alice），且 alice 不属于 hadoop 组**。

### 前提

Container executor **必须能够访问container需要的本地文件和目录**（比如配置文件，jars，共享对象等），应该设置为 755。

`yarn.nodemanager.local-dirs`和`yarn.nodemanager.log-dirs`必须是**NodeManger user（yarn）和 group（hadoop） 所拥有**。

- NodeManager如果不是以root用户启动，需要配置**/sys/fs/cgroup中的权限必须该用户可读写**；

### LinuxContainerExecutor

<code>LinuxContainerExecutor</code> 使用外部程序 `container-executor`启动container，可以执行`setuid`访问权限。

### 配置

```xml
<property>
	<name>yarn.nodemanager.linux-container-executor.path</name>
    <value></value>
    <description>container-executor的路径，默认在$HADOOP_HOME/bin/container-executor</description>
</property>
```

可执行文件`bin/container-executor`必须**为root用户所有，所属组必须为NodeManager user的用户组（hadoop，且不是应用用户所属的组）**，且**权限为 6050**。

**container-executor.cfg** 必须要为`root`所有，用户组任意，且权限设置为0400。

```ini
# configured value of yarn.nodemanager.linux-container-executor.group
yarn.nodemanager.linux-container-executor.group=hadoop
banned.users=root
# Prevent other super-users
min.user.id=1000
# comma separated list of allowed system users
allowed.system.users=yarn
# Terminal feature (feature.terminal.enabled) allows restricted shell into secure container via YARN UI2.
feature.terminal.enabled=1
```

系统还要求`etc/hadoop/container-executor.cfg` 的所有父目录(一直到`/` 目录) owner 都为 root。

```shell
# 重新编译放置container-executor.cfg 文件目录
mvn clean package -Dcontainer-executor.conf.dir=/etc/hadoop/ -DskipTests -Pnative
```

通过 `bin/container-executor --checksetup`，如果没有输出，表示配置成功。

