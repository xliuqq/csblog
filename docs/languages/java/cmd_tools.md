# Tools

**当前没有方法可以对 JNI 区分JVM使用的内存和C++使用的内存。**

- 进程内存可以通过 `/proc` 获取；
- JVM内存由 Heap + Meta + Direct Memory 等组成；

## 命令行

### java

`-server`：服务器模式启动JAVA，默认启动JVM基本都是-server模式

启用调试； -XDebug -Xrunjdwp:transport=dt_scoket, server=y,address=7899,suspend=n

- -Xrunjdwp 加载JDWP的JPDA参考执行实例；1.7之后可以将 -Xrunjdwp: 替换为 -agentlib:jdwp=
- transport：debugee与debuger调试时之间的通讯数据传输方式。 
- server：是否监听debuger的调试请求(作为调试服务器执行)。 
- suspend：是否等待启动，也即设置是否在debuger调试链接建立后才启动debugee JVM。 
- address： 调试服务器监听的端口号

**`-cp`**

- 指定**加载某路径下的所有第三方包**，-Djava.ext.dirs 或者 **java -cp .;c:\classes\myClass.jar;d:\classes\*.jar**

  linux下为 **java -cp .:/home/text/***，注意 * 不包括递归的子目录中的内容。

**OOM时堆快照**

-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${目录}

- 当JVM发生OOM时，自动生成DUMP文件。
- 如果不指定文件名，默认为：java_<pid>_<date>_<time>_heapDump.hprof

**`-jar`**

```java
java -jar myClass.jar
```

执行该命令时，会用到目录`META-INF\MANIFEST.MF`文件，在该文件中`Main-Class`的参数说明java -jar命令执行的类。

### Jar

解压

```css
jar -xvf jar包名字.jar
```

打包

```css
jar -cvfM 合并后的jar包名字.jar .
```





### javap（查看class文件)

>  javap -c -s -l -verbose $ClassName

- -c ：输出类中各方法的未解析的代码，即构成java字节码的指令；
- -l ：输出行及局部变量表；
- -s ：输出内部类型签名；
- -verbose ：打印堆栈大小、各方法的locals及args参数，以及class文件的编译版本；

### jps（JVM参数信息）

```shell
jps -m #输出传递给main方法的参数
jps -l  #输出应用程序main class的完整package名或者应用程序的jar文件完整路径名
jps -v #输出jvm参数
```

### jcmd（类，线程以及虚拟机信息）

 打印一个 Java 进程的类，线程以及虚拟机信息。使用 jcmd - h 来查看使用方法。

```shell
jcmd <pid | main class> <command ...|PerfCounter.print|-f file>

# 查看支持的命令列表，如 GC.heap_dump, VM.native_memory等
jcmd pid help
```

PerfCounter.print和jstat一样使用PerfData,jstat中的指标都可以根据这些counter计算出来，具体的计算规则可以参考tool.jar中的sun/tools/jstat/resources/jstat_options文件。

### jhat（html形式查看堆信息）

帮助分析内存堆存储，分析jmap输出的dump文件。

```shell
jhat dump.log
```

### jmap（内存信息，类实例数目和内存占用）

提供 JVM 内存使用信息，适用于脚本中。

```
-dump:[live,]format=b,file=<filename> 使用hprof二进制形式,输出jvm的heap内容到文件. live子选项是可选的，假如指定live选项,那么只输出活的对象到文件.

-finalizerinfo 打印正等候回收的对象的信息.

-heap 打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况.

-histo[:live] 打印每个class的实例数目,内存占用,类全名信息. VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量.

-clstats 打印classload和jvm heap长久层的信息. 包含每个classloader的名字,活泼性,地址,父classloader和加载的class数量. 另外,内部String的数量和占用内存数也会打印出来.(-clstats是-permstat的替代方案，在JDK8之前，-permstat用来打印类加载器的数据)
-F 强迫.在pid没有相应的时候使用-dump或者-histo参数. 在这个模式下,live子参数无效.
```

### jinfo（系统属性）

访问 JVM 系统属性，同时可以动态修改这些属性。

### jstack（堆栈和死锁信息）

提供 Java 进程内的线程堆栈信息。

### jstat（类加载，GC，编译信息）

提供 Java 垃圾回收以及类加载信息。

```shell
# S0：幸存1区当前使用比例 S1：幸存2区当前使用比例 E：伊甸园区使用比例 O：老年代使用比例 M：元数据区使用比例
# CCS：压缩使用比例 YGC：年轻代垃圾回收次数 FGC：老年代垃圾回收次数 FGCT：老年代垃圾回收消耗时间
# GCT：垃圾回收消耗总时间
jstat -gcutil pid

jstat -compiler pid

jstat -class pid
```

### jstatd（供远程attach监控本地进程）

jstatd是一个RMI的server应用，用于**监控jvm的创建和结束**，也为远程监视工具（如 jvisualvm）提供了一个可以attach的接口。 

- 配置 **jstatd.all.policy** 文件，安全策略

```shell
grant codebase "file:${java.home}/../lib/tools.jar" {
    permission java.security.AllPermission;
};
```

- 启动

```shell
jstatd -J-Djava.security.policy=jstatd.all.policy -J-Djava.rmi.server.hostname=172.16.2.132  -J-Djava.rmi.server.logCalls=true
```

### jmx和jstatd区别

JMX：使用JMX需要远**程JVM在启动的时候开启远程访问支持**，设定JMX端口等，每一个JMX连接一个远程JVM。
Jstatd：使用jstatd连接方式时，需要在**远程主机上创建安全策略文件然后启动jstatd进程**，并且此进程需要一直保持运行状态，客户端可以看到远程主机上当前用户的所有JVM的信息，即只要创建一个jstatd连接。



## 图形化

### jconsole(使用jvisualvm替代)

提供 JVM 活动的图形化展示，包括线程使用，类使用以及垃圾回收（GC）信息。

### jvisualvm（JVM分析，cpu/memory/gc/hot spot）

提供本地和远程 JVM 的可视化工具，剖析运行中的应用程序，分析 JVM 堆存储。

对于远程JVM，需要配置JMX或者远程启动jstatd：

```shell
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=10207 -Dcom.sun.management.jmxremote.
authenticate=false -Dcom.sun.management.jmxremote.ssl=false
```

GC通过插件Visual GC查看。

### jmc（分析性能）

飞行记录器：一般信息、内存、代码、线程、I/O、系统、事件。



## 案例

### idea远程debug

在 DEBUG 里面新建 Remote，

- Attach模式，远程开启端口，本地去连接。远程jvm启动参数中添加以下配置（jdk 8）：

```shell
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```

- Socket模式，本地开端口，远程启动时连接。远程jvm启动参数中添加以下配置：

```shell
-agentlib:jdwp=transport=dt_socket,server=n,address=DESKTOP-39BEHUR:5005,suspend=y,onthrow=<FQ exception class name>,onuncaught=<y/n>
```

### OutputMemory排查

```java
// 内存泄露代码
import java.util.ArrayList;
import java.util.List;

public class HeapDumpDemo {

    static class OOMHeapDumpObject {
        String str = "1234567890";
    }

    public static void main(String[] args) {
        List<OOMHeapDumpObject> ooms = new ArrayList<>();
        while (true) {
            ooms.add(new OOMHeapDumpObject());
        }
    }
}
```

如下启动：

```shell
java -XX:+HeapDumpOnOutOfMemoryError -Xmx20m HeapDumpDemo
```

将堆存储hprof文件，通过jvisualvm/idea等工具打开，可以查看到对应的大量的对象。