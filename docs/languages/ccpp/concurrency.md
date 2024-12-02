# 并发

## volatile

> java 中的 volatile 既保证可见性，还具备内存屏障（防重排序）的作用。

**编译器做的是静态乱序优化，CPU做的是动态乱序优化。**

- volatile关键字能够用来解决内存与I/O统一编址时，对硬件的即时访问。

```c
// volatile 仅保证可见性，关键字表示每次内存数据读取都走主存（不走寄存器），避免静态乱序优化
volatile 
// volatile 等于编译器级别屏障，不能阻止CPU硬件级别导致的重排
#define barrier() __asm__ __volatile__("": : :"memory") 
    
// CPU屏障 CPU Barrior：函数表示内存屏障/禁止指令重排序，避免动态乱序优化
__sync_synchronize()
// C++11为内存屏障提供了专门的函数std::atomic_thread_fence
```



## 线程本地变量

Static memory local to a thread (线程局部静态变量)，同时也可称之为线程特有数据（**TSD: Thread-Specific Data**）或者线程局部存储（**TLS: Thread-Local Storage**）。

```c++
#include <pthread.h>

// 确保无论有多少个线程调用多少次该函数，也只会执行一次由init所指向的由调用者定义的函数。
int pthread_once (pthread_once_t *once_control, void (*init) (void));

// Returns 0 on success, or a positive error number on error
int pthread_key_create (pthread_key_t *key, void (*destructor)(void *));

// Returns 0 on success, or a positive error number on error
int pthread_key_delete (pthread_key_t key);

// Returns 0 on success, or a positive error number on error
int pthread_setspecific (pthread_key_t key, const void *value);

// Returns pointer, or NULL if no thread-specific data is associated with key
void *pthread_getspecific (pthread_key_t key);
```



**只要线程终止时与key关联的值不为NULL，则destructor所指的函数将会自动被调用。**



智能的`__thread`的变量声明。