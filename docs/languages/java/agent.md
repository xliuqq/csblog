# Java Agent

> 运行时动态修改类，代码无侵入；

## 启动时加载(Premain)

通过 ***-javaagent*** 参数指定一个特定的 jar 文件（包含**Instrumentation 代理**）来启动代理程序；

```shell
java -javaagent:jar Instrumentation_jar -jar xxx.jar
```

`Instrumentation_jar`中的 **META-INF/MAINFEST.INF**文件的定义：

- **Premain-Class**：必须包含，**JVM启动时指定代理**，包含**premain**方法的类；

```java
// 先找该方法签名
public static void premain(final String args, final Instrumentation instrumentation) {}
// 找不到，再找该方法函数
public static void premain(String agentArgs);
```

## 运行时加载(Agentmain)

> 在主程序运行前就指定*javaagent*参数，*premain*方法中代码出现异常会导致主程序启动失败等，为了解决这些问题，JDK1.6以后提供了在**程序运行之后改变程序**的能力。

通过***attach***机制，将JVM A连接至JVM B，并**发送指令给JVM B执行**。

`agent.jar`中的**META-INF/MAINFEST.INF**文件

- **Agentmain-Class**：必须包含，**JVM启动时指定代理**，包含agentmain方法的类；

```java
public static void agentmain(final String args, final Instrumentation instrumentation) {}
public static void agentmain(String agentArgs);
```

JVM A 的示例代码：

```java
public static void main(String[] args) {
    try {
        String jvmPid = "23281"; // 目标进行的pid;
        VirtualMachine vm = VirtualMachine.attach(jvmPid);  
        vm.loadAgent("agent.jar");
        vm.detach();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```



## Instrument

instrument是JVM提供的一个可以修改已加载类的类库，专门为Java语言编写的插桩服务提供支持。它需要依赖JVMTI的Attach API机制实现。

- 实现它提供的ClassFileTransformer接口，定义一个类文件转换器。接口中的transform()方法会在类文件被加载时调用；
- 在**transform**方法里，我们可以利用上文中的 ASM 或 Javassist 对传入的字节码进行改写或替换，生成新的字节码数组后返回。



## 配置

- 在一个普通 Java 程序（带有 main 函数的 Java 类）运行时，通过 `-javaagent` 参数指定一个特定的 jar 文件（包含 Instrumentation 代理）来启动 Instrumentation 的代理程序。**在类的字节码载入jvm前会调用ClassFileTransformer的transform方法**，从而实现修改原类方法的功能，实现aop；
- **不会像动态代理或者cglib技术实现aop那样会产生一个新类**，也不需要原类要有接口；

```xml
<!-- maven 打包生成-->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <archive>
            <manifestEntries>
                <Agent-Class>com.uber.profiling.Agent</Agent-Class>
                <Premain-Class>com.uber.profiling.Agent</Premain-Class>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

- `META-INF/MAINFEST.INF`文件
  - `Premain-Class`：必须包含，JVM启动时指定代理，包含premain方法的类；
  - `Agent-Class`：支持VM启动之后，在某时刻启动代理的机制，指定代理类（包含agentmain方法）；
  - `Boot-Class-Path`：引导类加载器搜索的路径列表；
  - `Can-Redefine-Classes`：布尔值（默认），能够重定义此代理需要的类；
  - `Can-Retransform-Classes`：true或false（默认），能否重转换此代理需要的类；
  - `Can-Set-Native-Method-Prefix`：true 或 false（默认），是否能设置此代理所需的本机方法前缀；