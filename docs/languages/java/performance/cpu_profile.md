# CPU Profiler

CPU Profiling 有两种实现：**Sampling** 和 **Instrumentation**

- 基于无侵入的额外线程对所有线程的调用栈快照进行固定频率抽样，性能开销很低；
- 利用 Instrument API，对所有必要的 Class 进行字节码增强，性能开销高；

## Sampling

### Java Agent + JMX

#### 原理

基于对StackTrace的“采样”进行实现，如 [JVM Profile](./jvm_profile.md)

- 以 `Java Agent` 为入口，进入目标 JVM 进程后开启一个 `ScheduledExecutorService`，定时利用JMX的`threadMXBean.dumpAllThreads()` 来导出所有线程的 `StackTrace`，最终汇总并导出即可；
- `dumpAllThreads()`的执行开销不容小觑，Interval不宜设置的过小（[JVM Profile](./jvm_profile.md)默认为100ms）。

```java
public void profile() {
    ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
    // ...
    for (ThreadInfo threadInfo : threadInfos) {
        String threadName = threadInfo.getThreadName();
        // ...
        StackTraceElement[] stackTraceElements = threadInfo.getStackTrace();
        // ...
        for (int i = stackTraceElements.length - 1; i >= 0; i--) {
            StackTraceElement stackTraceElement = stackTraceElements[i];
            // ...
        }
        // ...
    }
}
```

#### 优劣势

优势：

- **性能开销很低**：相比于Instrumentation为几乎所有方法添加额外AOP逻辑；

劣势：

- JVM固有的**只能在安全点（Safe Point）进行采样的“缺陷”**，会导致统计结果存在一定的偏差；

- Java Agent代码与业务代码共享AppClassLoader，被JVM直接加载的agent.jar如果引入了第三方依赖，可能会**对业务Class造成污染**；

### JVMTI + GetStackTrace

> 解决对业务Class造成污染的问题。

- 分离入口与核心代码，使用定制的ClassLoader加载核心代码，避免影响业务代码；
- 直接对接JVMTI接口，使用原生C API对JVM进行操作；

#### 原理

1. 编写Agent_OnLoad()，在入口通过JNI的JavaVM*指针的GetEnv()函数拿到JVMTI的jvmtiEnv指针：

```c
// agent.c
#include <jvmti.h>
JNIEXPORT jint JNICALL Agent_OnLoad(JavaVM *vm, char *options, void *reserved) {
    jvmtiEnv *jvmti;
    (*vm)->GetEnv((void **)&jvmti, JVMTI_VERSION_1_2);
    // ...
    return JNI_OK;
}
```

2. 开启一个线程定时循环，定时使用 jvmtiEnv 指针配合调用如下几个**JVMTI函数**：

```c
// 获取所有线程的jthread
jvmtiError GetAllThreads(jvmtiEnv *env, jint *threads_count_ptr, jthread **threads_ptr);

// 根据jthread获取该线程信息（name、daemon、priority...）
jvmtiError GetThreadInfo(jvmtiEnv *env, jthread thread, jvmtiThreadInfo* info_ptr);

// 根据jthread获取该线程调用栈
jvmtiError GetStackTrace(jvmtiEnv *env,
                         jthread thread,
                         jint start_depth,
                         jint max_frame_count,
                         jvmtiFrameInfo *frame_buffer,
                         jint *count_ptr);
```

#### 优劣势

优势

- Java Agent + JMX 的优势；
- 不对业务Class造成污染；

劣势：

- 开发效率偏低；
- **只能在安全点（Safe Point）进行采样的“缺陷”**，会导致统计结果存在一定的偏差；



### JVMTI + AsyncGetCallTrace

> 解决**只能采集到位于安全点时刻的调用栈快照**的问题。

- **JVMTI的GetStackTrace()函数不需要在Caller的安全点执行，但当调用GetStackTrace()获取其他线程的调用栈时，必须等待，直到目标线程进入安全点**；
- **GetStackTrace()仅能通过单独的线程同步定时调用，不能在UNIX信号处理器的Handler中被异步调用**；

开源实现：[Async-Profiler](#Async-Profiler)

#### 原理

AsyncGetCallTrace

- 获取当前线程的调用栈且不受安全点干扰，还支持在UNIX信号处理器中被异步调用；
- **UNIX信号会被发送给进程的随机一线程进行处理**，因此最终信号会均匀分布在所有线程上，也就均匀获取了所有线程的调用栈样本。

```c
// 栈帧
typedef struct {
 jint lineno;
 jmethodID method_id;
} AGCT_CallFrame;

// 调用栈
typedef struct {
    JNIEnv *env;
    // 正常情况下标识了获取到的调用栈深度，但在Native代码执行期间、GC期间下它就表示为负数，最常见的-2代表此刻正在GC
    jint num_frames;
    AGCT_CallFrame *frames;
} AGCT_CallTrace;

// 根据ucontext将调用栈填充进trace指针
void AsyncGetCallTrace(AGCT_CallTrace *trace, jint depth, void *ucontext);
```

AsyncGetCallTrace非标准JVMTI函数，不在jvmti.h中声明：

- 在Agent_OnLoad内通过glibc提供的**dlsym()**函数拿到当前地址空间（即目标JVM进程地址空间）名为“AsyncGetCallTrace”的符号地址；
- 对符号地址（即函数指针）按照函数原型进行转换即可；

流程：

- 入口拿到jvmtiEnv和AsyncGetCallTrace指针，获取AsyncGetCallTrace方式如下

```c
typedef void (*AsyncGetCallTrace)(AGCT_CallTrace *traces, jint depth, void *ucontext);
// ...
AsyncGetCallTrace agct_ptr = (AsyncGetCallTrace)dlsym(RTLD_DEFAULT, "AsyncGetCallTrace");
if (agct_ptr == NULL) {
    void *libjvm = dlopen("libjvm.so", RTLD_NOW);
    if (!libjvm) {
        // 处理dlerror()...
    }
    agct_ptr = (AsyncGetCallTrace)dlsym(libjvm, "AsyncGetCallTrace");
}
```

- 在OnLoad阶段，我们还需要做一件事，即注册OnClassLoad和OnClassPrepare这两个Hook，**原因是jmethodID是延迟分配的，使用AGCT获取Traces依赖预先分配好的数据。我们在OnClassPrepare的CallBack中尝试获取该Class的所有Methods，这样就使JVMTI提前分配了所有方法的jmethodID**

```c
void JNICALL OnClassLoad(jvmtiEnv *jvmti, JNIEnv* jni, jthread thread, jclass klass) {}

void JNICALL OnClassPrepare(jvmtiEnv *jvmti, JNIEnv *jni, jthread thread, jclass klass) {
    jint method_count;
    jmethodID *methods;
    jvmti->GetClassMethods(klass, &method_count, &methods);
    delete [] methods;
}

// ...

jvmtiEventCallbacks callbacks = {0};
callbacks.ClassLoad = OnClassLoad;
callbacks.ClassPrepare = OnClassPrepare;
jvmti->SetEventCallbacks(&callbacks, sizeof(callbacks));
jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_CLASS_LOAD, NULL);
jvmti->SetEventNotificationMode(JVMTI_ENABLE, JVMTI_EVENT_CLASS_PREPARE, NULL);
```

- 利用SIGPROF信号来进行定时采样：

```c
// 这里信号handler传进来的的ucontext即AsyncGetCallTrace需要的ucontext
void signal_handler(int signo, siginfo_t *siginfo, void *ucontext) {
    // 使用AsyncCallTrace进行采样，注意处理num_frames为负的异常情况
}

// ...

// 注册SIGPROF信号的handler
struct sigaction sa;
sigemptyset(&sa.sa_mask);
sa.sa_sigaction = signal_handler;
sa.sa_flags = SA_RESTART | SA_SIGINFO;
sigaction(SIGPROF, &sa, NULL);

// 定时产生SIGPROF信号
// interval是nanoseconds表示的采样间隔，AsyncGetCallTrace相对于同步采样来说可以适当高频一些
long sec = interval / 1000000000;
long usec = (interval % 1000000000) / 1000;
struct itimerval tv = {{sec, usec}, {sec, usec}};
setitimer(ITIMER_PROF, &tv, NULL);
```

- 在Buffer中保存每一次的采样结果，最终生成必要的统计数据即可。

#### 优劣势

优势：

- 社区中目前性能开销最低、相对效率最高的CPU Profiler实现方式；
- 结合perf_events还能做到同时采样Java栈与Native栈，也就能同时分析Native代码中存在的性能热点；



## [Async-Profiler](https://github.com/jvm-profiling-tools/async-profiler)

- 支持基于 HotSpot JVM 的OpenJDK, Oracle JDK；

**CPU Profiling**

> stack trace samples that include **Java** methods, **native** calls, **JVM** code and **kernel** functions.

- 基于`perf_events` + `AsyncGetCallTrace`

**ALLOCATION profiling**

- 不适用代价大的bytecode instrumentation 或者 DTrace probes；
- 不影响逃逸分析和 JIT 优化；
- 依赖于 HotSpot 特定的callbacks，基于 TLAB 驱动采样；

### Agent Launching

```
$ java -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.html ...
```



### 命令

#### profiler.sh

```shell
$ ./profiler.sh -o collapsed -f /tmp/traces-%t.txt 8983
```



#### Jattach 

- 通过 Java Agent 的 `AgentMain`方法实现向[运行中的JVM进程添加Agent](../agent.md)；
- 对于JVMTI，需实现一个Agent_OnAttach()函数，当将JVMTI Agent Attach到目标进程时，从该函数开始执行；

以Async-Profiler中的jattach源码为线索，探究一下如何**利用Attach机制给运行中的JVM进程发送命令**。

```shell
# libagent.so就被加载到ID为1234的JVM进程中并开始执行Agent_OnAttach函数
# 执行Attach的进程euid及egid，与被Attach的目标JVM进程必须相同
$ jattach 1234 load /absolute/path/to/agent/libagent.so true
```

[原理（相关代码）](https://github.com/jvm-profiling-tools/async-profiler/blob/master/src/jattach/jattach_hotspot.c)：

- HotSpot 提供一种特殊的机制，只要给它发送一个SIGQUIT信号，并预先准备好.attach_pid文件，HotSpot会主动创建一个地址为“/tmp/.java_pid”的UNIX Socket，接下来主动Connect这个地址即可建立连接执行命令；

- 外部进程与目标JVM进程之间发送的数据格式相当简单，基本如下所示：

  ```shell
  <PROTOCOL VERSION>\0<COMMAND>\0<ARG1>\0<ARG2>\0<ARG3>\0
  ```

  以先前我们使用的Load命令为例，发送给HotSpot时格式如下：

  ```shell
  1\0load\0/absolute/path/to/agent/libagent.so\0true\0\0
  ```