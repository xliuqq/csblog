## Ping-Pong Buffer

ping-pong buffer 也叫**双缓存 double buffer**，(必须是两个) 就是一个缓存在写入的时候，另一个缓存同时在处理的结构。

- 最常见的例子是显示缓存可以用 ping-pong buffer 结构, 用两个相同的帧存储器 frame buffers, 当一个在扫描输出的时候，另一个在改写数据。到下一个周期改写好数据的一帧作扫描输出,，而另一个帧缓存改写数据。总之是一个在接收数据一个在处理，处理完后两个再交换。

- **存在写阻塞问题**，所以一般用于“一写多读”配置热更新场景，且写线程往往是异步的对阻塞时长容忍度高。

## 双缓冲队列

在服务器开发中 通常的做法是把逻辑处理线程和IO处理线程分离。 

- I/O 处理线程负责网络数据的发送和接收，连接的建立和维护。 
- 逻辑处理线程处理从IO线程接收到的包。

通常逻辑处理线程和IO处理线程是通过数据队列来交换数据，就是生产者--消费者模型。

双缓冲数据就是两个队列：一个负责从里写入数据，一个负责读取数据。当逻辑线程读完数据后负责将自己的队列和IO线程的队列进行交换。

这种操作需要在两个地方加锁：

- IO线程向队列写入数据。
- 两个队列进行交换时。

双缓冲区节省了**单缓冲区时读部分操作互斥/同步**的开销。

两个缓冲区分别对应着两个互斥锁LockA，LockB。