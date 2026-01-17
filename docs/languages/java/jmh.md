# JMH

> Java Microbenchmark Harness，它是由 **Java 虚拟机团队**开发的一款用于 Java **微基准测试工具**。

JMH是Open JDK9自带的，如果你是 JDK9 之前的版本也可以通过导入 openjdk

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.23</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.23</version>
</dependency>
```

示例：运行 JMH 基准测试的**推荐方法**是使用Maven设置一个独立的项目，该项目依赖于应用程序的jar文件。

```java
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

@BenchmarkMode(Mode.AverageTime)
// 每个进行基准测试的线程都会独享一个对象示例。
@State(Scope.Thread)
// 表示开启一个线程进行测试
@Fork(1)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
// 微基准测试前进行三次预热执行
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class JmhHello {

    String string = "";
    StringBuilder stringBuilder = new StringBuilder();

    @Benchmark
    public String stringAdd() {
        for (int i = 0; i < 1000; i++) {
            string = string + i;
        }
        return string;
    }

    @Benchmark
    public String stringBuilderAppend() {
        for (int i = 0; i < 1000; i++) {
            stringBuilder.append(i);
        }
        return stringBuilder.toString();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
            .include(JmhHello.class.getSimpleName())
            .build();
        new Runner(opt).run();
    }
}

```

