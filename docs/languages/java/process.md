# 进程

> 测试代码见[代码库](https://gitee.com/oscsc/jvm/tree/master/process)

## 子进程

> Process blocking reason: **The input stream and the error stream are separated, and if they are not processed, blocking will occur**.

- 创建的子进程的所有标准 io（即 stdin，stdout，stderr）操作通过三个流 (getOutputStream()，getInputStream()，getErrorStream()) 重定向到父进程；
- 有些本机平台仅针对标准输入和输出流提供有限的缓冲区大小，如果读写子进程的输出流或输入流迅速出现失败，则可能导致子进程阻塞，甚至产生死锁；
- 标准输出流和错误流如果处理不当，其**缓冲区不能被及时清除而被塞满，则进程被阻塞**，即使调用Process.destory()也未必能销毁被阻塞的子进程



解决方案：

- 并发获取Process的输出流和错误流；
- 将错误流重定向到输出流；



### 环境变量

默认是继承 Java 进程的环境变量（等价于 System.getEnv）



### 父进程退出时Kill子进程

> 通过ProcessBuilder启动JVM，并注册kill父进程时执行的线程(kill子线程)；

通过ProcessBuilder执行shell命令(如再启动java进程），此时通过jps查看会看到两个java进程；**如果kill掉父进程，子进程不会随之消亡**，若需要，需注册（见下面代码）；

```java
public class TestRunJVM {
    public static void main(String[] args) {
        ProcessBuilder pb = new ProcessBuilder("jvisualvm");
        try {
            final Process process = pb.start();
            // destroy child process，对于 kill -9 不生效
            Runtime.getRuntime().addShutdownHook(new Thread() {
                public void run() {
                    if (process != null) {
                        process.destroy();
                        try {
                            process.waitFor();
                        } catch (Exception e){
                            System.out.println("ERROR2" + e.toString());
                        }
                    }
                }
            });
            process.waitFor();
        } catch (Exception e) {
            System.out.println("ERROR" + e.toString());
        }
    }
}
```

 