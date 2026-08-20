# Yarn Application

## 原生

> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/WritingYarnApplications.html

### 接口类

- **Client ** <--> **ResourceManager** ：`YarnClient` 

- **ApplicationMaster**<-->**ResourceManager**

  - 通过 `AMRMClientAsync` 对象，通过 `AMRMClientAsync.CallbackHandler` 异步处理事件；
  - `AMRMClient` 对象，通过 `allocate` 函数同步获取分配的container；

- **ApplicationMaster**<-->**NodeManager**

  启动Container，通过 `NMClientAsync` 和NodeManager通信，通过 `NMClientAsync.CallbackHandler`处理container事件


**更底层的API，一般不推荐使用**

ApplicationClientProtocol ：**Client ** <--> **ResourceManager**

ApplicationMasterProtocol ：**ApplicationMaster**<-->**ResourceManager**

ContainerManagementProtocol ：**ApplicationMaster**<-->**NodeManager**

### Client

```java
// 启动YarnClient
YarnClient yarnClient = YarnClient.createYarnClient();
yarnClient.init(conf);
yarnClient.start();
// 创建应用，获取应用id
YarnClientApplication app = yarnClient.createApplication();
GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
ApplicationId appId = appContext.getApplicationId();

// 构建 AM Container
ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(localResources, env, commands, null, null, null);
appContext.setAMContainerSpec(amContainer);

// 提交应用
yarnClient.submitApplication(appContext);
```

### AM

```java
String containerIdString = envs.get(ApplicationConstants.AM_CONTAINER_ID_ENV);
// 异步Client，配置 CallBack，处理 onContainersAllocated 
AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
amRMClient.init(conf);
amRMClient.start();

containerListener = createNMCallbackHandler();
nmClientAsync = new NMClientAsyncImpl(containerListener);
nmClientAsync.init(conf);
nmClientAsync.start();

// 注册 AM
appMasterHostname = NetUtils.getHostname();
RegisterApplicationMasterResponse response = amRMClient.registerApplicationMaster(appMasterHostname, appMasterRpcPort,
        appMasterTrackingUrl);

List<Container> previousAMRunningContainers =  response.getContainersFromPreviousAttempts();
int numTotalContainersToRequest =
    numTotalContainers - previousAMRunningContainers.size();
// 请求 RM 分配 Container
for (int i = 0; i < numTotalContainersToRequest; ++i) {
  // 构建 ContainerRequest 
  ContainerRequest containerAsk = setupContainerAskForRM();
  amRMClient.addContainerRequest(containerAsk);
}
// Keep looping until all the containers are launched and shell script executed on them ( regardless of success/failure).
```



```java
@Override
public void onContainersAllocated(List<Container> allocatedContainers) {
  LOG.info("Got response from RM for container ask, allocatedCnt="
      + allocatedContainers.size());
  numAllocatedContainers.addAndGet(allocatedContainers.size());
  for (Container allocatedContainer : allocatedContainers) {
    // 构建 ContainerLaunchContext 
    LaunchContainerRunnable runnableLaunchContainer =
        new LaunchContainerRunnable(allocatedContainer, containerListener);
    Thread launchThread = new Thread(runnableLaunchContainer);

    // launch and start the container on a separate thread to keep
    // the main thread unblocked
    // as all containers may not be allocated at one go.
    launchThreads.add(launchThread);
    launchThread.start();
  }
}
```



### Unmanaged AM

> YARN引入了一种新的ApplicationMaster—Unmanaged AM，这种AM运行在客户端，不再由ResourceManager启动和销毁（即 RM 不分配 AM资源）。

Unmanaged AM运行步骤如下：（具体可参考 Spark 对 Unmanaged AM 的使用流程）

- 通过RPC函数`ClientRMProtocol.getNewApplication()`获取一个ApplicationId
- 创建一个`ApplicationSubmissionContext`对象，填充各个字段，并通过调用函数`ApplicationSubmissionContext.setUnmanagedAM(true)`启用Unmanaged AM。
- 通过RPC函数`ClientRMProtocol.submitApplication()`将application提交到ResourceManage上，并监控application运行状态，直到其状态变为`YarnApplicationState.ACCEPTED`。
- 在客户端中的一个独立线程中启动 ApplicationMaster，然后等待ApplicationMaster运行结束，接着再等待ResourceManage报告application运行结束。



## 第三方[skein](https://github.com/jcrist/skein)

> A tool(cli) and library (python) for easily deploying applications on Apache YARN.

### 原理

Java 写的 Driver 服务，作为 Yarn Client 跟 Yarn RM通信；

Java 写的 Application Master 服务：

- 负责解析 Skin App Yaml，向Yarn申请/释放Container，同时可以运行一个用户定义的进程（application driver）；
- 包含 [KV Store](https://jcristharif.com/skein/key-value-store.html)，可用于container间信息共享或者协调状态等；

### 规范

支持 Master 和 Service；

### 使用

1. 安装

```shell
$ conda install -c conda-forge skein
```

2. 启动 driver，全局driver，如果不启动全局，则每次提交时会启动临时的driver；

```shell
$ skein driver start
```

3. 写应用程序定义

```yaml
name: hello_world
queue: default

master:
  resources:
    vcores: 1
    memory: 512 MiB
  script: |
    sleep 60
    echo "Hello World!"
```

4. 提交程序

```shell
$ skein application submit hello_world.yaml
```

