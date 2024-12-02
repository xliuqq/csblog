# Java NIO

Java中跟zero copy相关的主要集中在FileChannel和MappedByteBuffer中。

网络通讯框架Netty4中跟zero copy相关的则主要集中在FileRegion和CompositeByteBuf中。



## 缓冲区 Buffer

绍缓冲区（Buffer） 是一个对象，它包含一些要写入或者要读出的数据。

```java
ByteBuffer buf = ByteBuffer.allocate(1024);
```

- 分配的是**HeapByteBuffer**，被JVM管理，当**GC时其 buf 在内存中的实际地址会发生变化**；
- 当进行 IO 操作时，需要转换为 DirectByteBuffer，防止出现GC时 JVM 对其内存地址发生变化；



### MappedByteBuffer

java nio提供的FileChannel提供`map()`方法，该方法可以在一个`打开的文件和MappedByteBuffer`之间建立一个虚拟内存映射，`MappedByteBuffer`继承于`ByteBuffer`，类似于一个基于内存的缓冲区，只不过该对象的数据元素存储在磁盘的一个文件中；

**示例代码**

```java
public class MappedByteBufferTest {
    public static void main(String[] args) throws Exception {
        File file = new File("D://db.txt");
        long len = file.length();
        byte[] ds = new byte[(int) len];
        MappedByteBuffer mappedByteBuffer = new FileInputStream(file).getChannel().map(FileChannel.MapMode.READ_ONLY, 0, len);
        for (int offset = 0; offset < len; offset++) {
            byte b = mappedByteBuffer.get();
            ds[offset] = b;
        }
        Scanner scan = new Scanner(new ByteArrayInputStream(ds)).useDelimiter(" ");
        while (scan.hasNext()) {
            System.out.print(scan.next() + " ");
        }
    }
}
```

MappedByteBuffer本身是一个抽象类，其实这里真正实例是DirectByteBuffer。



### DirectByteBuffer

> `unsafe.allocateMemory(size)` native方法，通过C的malloc来进行分配（`brk`系统调用）；

`DirectByteBuffer`：堆外内存，继承于MappedByteBuffer，不是JVM堆内存

- 直接的文件拷贝操作，或者I/O操作。直接使用堆外内存就能少去内存从用户内存拷贝到系统内存的操作

```java
ByteBuffer directByteBuffer = ByteBuffer.allocateDirect(100);
```



**C 和 Java中进行 DirectBuffer 共享使用**：

> 示例代码：[direct memory在C/Java共享](https://gitee.com/oscsc/jvm/tree/master/directmemory)

```java
import java.nio.ByteBuffer;

public class DirectBufferExample {
    public static native void processBuffer(ByteBuffer buffer);

    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
        processBuffer(buffer);
    }

    static {
        System.loadLibrary("example");
    }
}
```



```c
#include <jni.h>

JNIEXPORT void JNICALL Java_DirectBufferExample_processBuffer(JNIEnv *env, jobject obj, jobject buffer) {
    char* data = (char*) (*env)->GetDirectBufferAddress(env, buffer);
    jint capacity = (*env)->GetDirectBufferCapacity(env, buffer);

    // 访问缓冲区并进行处理
    for (int i = 0; i < capacity; i++) {
        data[i] = (char) (i % 256);
    }
}
```



## 通道 Channel

Channel 是一个通道，可以通过它读取和写入数据。

通道与流的不同之处在于通道是双向的，流只是在一个方向上移动，而且**通道可以用于读、写或者同时用于读写**。



### Channel-to-Channel传输

经常需要从一个位置将文件传输到另外一个位置，FileChannel提供了transferTo()方法用来提高传输的效率：

- **避免数据在缓冲区的拷贝开销**；

```java
public class ChannelTransfer {
    public static void main(String[] argv) throws Exception {
        String files[]=new String[1];
        files[0]="D://db.txt";
        catFiles(Channels.newChannel(System.out), files);
    }

    private static void catFiles(WritableByteChannel target, String[] files)
            throws Exception {
        for (int i = 0; i < files.length; i++) {
            FileInputStream fis = new FileInputStream(files[i]);
            FileChannel channel = fis.getChannel();
            // 重点
            channel.transferTo(0, channel.size(), target);
            channel.close();
            fis.close();
        }
    }
}
```



## 多路复用器 Selector

Selector 会不断地轮询注册在其上的 Channel，如果某个 Channel 上面有新的 TCP 连接接入、读和写事件，这个 Channel 就处于就绪状态，会被 Selector 轮询出来，然后通过 SelectionKey 可以获取就绪 Channel 的集合，进行后续的 I/O 操作。



## Netty中零拷贝

Zero拷贝的细节，见[该文档](../../cs/linux/zero_copy.md)。

### 避免数据流经用户空间

即[OS中的零拷贝]([该文档](../../os/IO.md#Linux OS的零拷贝（发送文件为例）))：

- Netty的文件传输调用**FileRegion包装的transferTo**方法，可以直接将[文件缓冲区](https://www.zhihu.com/search?q=文件缓冲区&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A76059333})的数据发送到目标Channel，避免通过循环write方式导致的内存拷贝问题，FileRegion底层调用**NIO FileChannel**的transferTo函数；

### 避免数据从JVM Heap到C Heap的拷贝

在JVM层面，每当程序需要执行一个I/O操作时，都需要将数据先从**JVM管理的堆内存**复制到**使用C malloc()或类似函数分配的Heap内存**中才能够触发系统调用完成操作，这部分内存站在Java程序的视角来看就是堆外内存，但是**以操作系统的视角来看其实都属于进程的堆区**

- Netty的接收和发送ByteBuffer使用**直接内存进行Socket读写**，不需要进行**字节缓冲区**的二次拷贝。
  - 如果使用JVM的堆内存进行Socket读写，JVM会将堆内存Buffer拷贝一份到直接内存中，然后才写入Socket中。相比于使用直接内存，消息在发送过程中多了一次缓冲区的内存拷贝；

### 减少数据在用户空间的多次拷贝

- Netty提供CompositeByteBuf类，可以将多个ByteBuf合并为一个**逻辑上的ByteBuf**, 避免了各个ByteBuf之间的拷贝；
- 通过**wrap操作**, 我们可以将byte[]数组、ByteBuf、ByteBuffer等包装成一个Netty ByteBuf对象, 进而避免拷贝操作；
- ByteBuf支持slice操作，可以将ByteBuf分解为**多个共享同一个存储区域的ByteBuf**，避免内存的拷贝。