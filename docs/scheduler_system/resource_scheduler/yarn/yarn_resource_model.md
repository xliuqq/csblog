

# Yarn资源模型和自定义设备插件

## Resource Model

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/ResourceModel.html

YARN 支持可扩展的资源模型。**默认情况下，YARN会跟踪所有节点，应用程序和队列的CPU和内存**，但资源定义可以扩展为包含任意 “countable” 资源。

### 自定义资源类型

> 针对普通的资源类型，不需要被 CGroup 隔离。

**resource-types.xml**（所有节点都要配置）

- 参数：`yarn.resource-types`，逗号分隔的附加资源列表；
- 参数：`yarn.resource-types.<resource>.units`，指定资源类型的默认单位；
- 参数：`yarn.resource-types.<resource>.minimum-allocation`，指定资源类型的最小请求；
- 参数：`yarn.resource-types.<resource>.maximum-allocation`，指定资源类型的最大请求；

**node-resources.xml**（所有NM节点都要配置）

- 参数：`yarn.nodemanager.resource-type.<resource>`，节点管理器中可用的指定资源的数量；

### Resource Profiles

#### 配置

**yarn-site.xml**

- `yarn.resourcemanager.resource-profiles.enabled`，是否启用资源配置文件支持，默认为 false；
- `yarn.resourcemanager.resource-profiles.source-file`：配置文件路径，支持 json；

**resource-profiles.json**

- minimum 和 maximum 是不可以被定义，在`resource-types.xml`中被定义；

```xml
{
    "small": {
        "memory-mb" : 1024,
        "vcores" : 1
    },
    "default" : {
        "memory-mb" : 2048,
        "vcores" : 2
    },
    "large" : {
        "memory-mb": 4096,
        "vcores" : 4
    },
    "compute" : {
        "memory-mb" : 2048,
        "vcores" : 2,
        "gpu" : 1
    }
}
```

#### 请求

申请 AM 时不支持直接使用 Resource Profile。

- 可以通过 YarnClient 获取特定的 Resource Profile，转化为 Resource，构建 ResourceRequest；

AM 向 RM 申请 Container 时；

- **ContainerRequest**：可同时指定 Resource 或者 ResourceProfile Name
  - Resource 中的资源会覆盖 Resource Profile 中的定义；
- **SchedulingRequest**：不支持 Resource Profile；

## 自定义设备插件

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/PluggableDeviceFramework.html

 插件式的适配支持：用于需要被 Cgroup 隔离的设备，提供通用的解决方案

- 依赖 `LinuxContainerExecutor ` 进行 Cgroup 隔离

### 配置

`capacity-scheduler.xml`

```xml
<property>
  <name>yarn.scheduler.capacity.resource-calculator</name>
  <value>org.apache.hadoop.yarn.util.resource.DominantResourceCalculator</value>
</property>
```

`yarn-site.xml`：开启设备插件框架

```xml
<property>
  <name>yarn.nodemanager.pluggable-device-framework.enabled</name>
  <value>true</value>
</property>
```

自定义设备资源和处理的类：

```xml
<property>
  <name>yarn.resource-types</name>
  <value>nvidia.com/gpu</value>
</property>
<property>
  <name>yarn.nodemanager.pluggable-device-framework.device-classes</name> 		     <value>org.apache.hadoop.yarn.server.nodemanager.containermanager.resourceplugin.com.nvidia.NvidiaGPUPluginForRuntimeV2</value>
</property>
```

通过`yarn node -list -showDetails`可以查看设备资源信息。



### 开发插件

#### 插件原理



#### Demo

maven 依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-yarn-server-nodemanager</artifactId>
      <version>3.3.0</version>
      <scope>provided</scope>
  </dependency>
</dependencies>
```

