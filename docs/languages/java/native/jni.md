# JNI(Java Native Interface)

> 基于[JDK21](https://java.cunzaima.cn/jdk21/doc-zh/specs/jni/invocation.html)，示例见 [Demo](./jni_demo.md)。

何时需要使用Java本地方法：

- 标准Java类库不支持应用程序所需的平台相关功能。
- 您已经有一个用另一种语言编写的库，并希望通过JNI使其可供Java代码访问。
- 您想要在较低级别语言（如汇编）中实现一小部分时间关键代码。

**JNI_OnLoad可以和JNIEnv的registerNatives函数结合起来，实现动态的函数替换。**

## 设计概述

### JNI接口函数和指针(JNIEnv*)

本地代码通过调用JNI函数访问Java虚拟机功能。JNI函数通过一个***接口指针***提供。

- 接口指针是一个指向指针的指针。该指针指向一个指针数组，每个指针指向一个接口函数。每个接口函数都位于数组内的预定义偏移量处。

![接口指针](.pics/jni/interface-pointer.gif)

JNI接口指针<font color='red'>**仅在当前线程**</font>中有效。因此，本地方法不得将接口指针从一个线程传递到另一个线程。

```c
/* 获取Java字符串的C副本 */
const char *str = (*env)->GetStringUTFChars(env, s, 0);

/** C++ 源代码中消失了额外的间接级别和接口指针参数 */
const char *str = env->GetStringUTFChars(s, 0);
```



### 编译、加载和链接本地方法

> 由于Java虚拟机是多线程的，本地库也应使用**支持多线程的本地编译器**进行编译和链接。
>
> - GCC 的 -D_REENTRANT

本地方法通过`System.loadLibrary`方法加载。虚拟机在内部为**每个类加载器**维护一个已加载本地库的列表。

- Linux系统将名称`p_q_r_A`转换为`libp_q_r_A.so`，而Windows系统将相同的`p_q_r_A`名称转换为`p_q_r_A.dll`。

```java
package p.q.r;

class A {
    native double f(int i, String s);
    static {
        System.loadLibrary("p_q_r_A");
    }
}
```

本机方法和接口API都遵循给定平台上的标准库调用约定。例如，UNIX系统使用C调用约定，而Win32系统使用__stdcall。

#### 本机方法参数

JNI接口指针是本机方法的第一个参数。JNI接口指针的类型为*JNIEnv*。

第二个参数取决于本机方法是静态的还是非静态的。非静态本机方法的第二个参数是对对象的引用。静态本机方法的第二个参数是对其Java类的引用。

其余参数对应于常规Java方法参数。

```c
jdouble Java_p_q_r_A_f__ILjava_lang_String_2 (
     JNIEnv *env,        /* 接口指针 */
     jobject obj,        /* "this"指针 */
     jint i,             /* 参数＃1 */
     jstring s)          /* 参数＃2 */   
```



### 引用Java对象

> 原始类型，如整数、字符等，在Java和本机代码之间进行复制。另一方面，任意Java对象是通过引用传递的。虚拟机必须跟踪所有传递给本机代码的对象，以便这些对象不会被垃圾回收器释放。

JNI 支持3中不透明的引用：**局部(local)引用**、**全局(global)引用**和**弱全局引用**。

- 局部引用在本机方法调用期间有效，并在<font color='red'>**本机方法返回后**</font>自动释放，局部引用仅在**创建它们的线程中**有效。
- 全局引用保持有效，直到显式释放为止。从局部引用创建全局引用。

对象作为**局部引用传递给本机**方法。所有由 **JNI 函数返回的Java对象都是局部**引用。

- 本机方法可以将局部或全局引用作为其结果返回给虚拟机。

有时程序员应显式释放局部引用：

- 在其余计算中不再使用该对象时，大的Java对象的局部引用将阻止垃圾回收对象；
- 本机方法创建大量局部引用，尽管并非所有引用同时使用，创建太多局部引用可能导致系统内存不足。

弱引用对象

- 一个局部或者全局引用，使所提及的对象不能被垃圾回收。而弱全局引用，则允许提及的对象进行垃圾回收。

释放引用：

- **基本数据类型是不需要释放**，如 jint , jlong , jchar 等等。
- 需要释放的是引用数据类型，当然也包括数组。如：jstring, jobject, j***Array, jclass 等。
- `jmethodID`和`jfieldID`不是引用类型，不需要被释放；

局部引用默认只有当**本地函数返回Java（当Java调用native）**或**调用线程detach JVM（native调用Java）**时才会被GC。

<font color='red'>对于C++ call Java时创建的 global reference和 local reference 创建，需要定义其释放之处；</font>

> 注：HotSpotVM：-XX:MaxJNILocalCapacity flag (default: 65536)。当前没有测试出来，Local references 溢出的情况；(JDK 8)



### Java异常

JNI允许本地方法引发任意Java异常。本地代码也可以处理未处理的Java异常。未处理的Java异常将传播回VM。

**本地代码中调用某个JNI接口时如果发生了异常，后续的本地代码不会立即停止执行，而会继续往下执行后面的代码。**

```cpp
// 如果这里出现问题，则会出现问题，后续执行异常
jauthority = env.newStringUTF(authority, "authority"); 
env->DeleteLocalRef(jauthority);
```

#### 异常和错误代码

在大多数情况下，JNI函数通过返回错误代码（特殊的返回值，如NULL）*并*抛出Java异常来报告错误条件。因此，程序员可以：

- 快速检查**最后一个JNI调用的返回值**，以确定是否发生错误（为NULL），并
- 调用一个函数`ExceptionOccurred()`，以获取包含错误条件更详细描述的异常对象。

有两种情况需要程序员先检查异常而不是先检查返回值：

- 通过 JNI 调用 Java 方法时（call jmethod, 可能返回结果就是NULL）；
- JNI数组访问函数不反悔错误代码，但可能抛出`ArrayIndexOutOfBoundsException`或`ArrayStoreException`。

在所有其他情况下，非错误返回值保证未引发异常。

#### 异步异常

在多个线程的情况下，当前线程以外的线程可能会发布异步异常。异步异常不会立即影响当前线程中本机代码的执行，直到：

- 本机代码调用可能引发同步异常的 JNI 函数，或者
- 机代码使用 `ExceptionOccurred（）` 显式检查同步和异步异常

请注意，只有那些可能引发同步异常的 JNI 函数才会检查异步异常。

#### 异常处理

在本地代码中处理异常有两种方式：

- 本地方法可以选择立即返回，导致**异常在启动本地方法调用的Java代码中**抛出。
- 本地代码可以通过**调用`ExceptionClear()`清除异常**，然后执行自己的异常处理代码。

libhdfs 中的标准用法
```c
// 封装了JNI 函数，在调用 JNI 函数后，通过 getPendingExceptionAndClear 返回异常。
jthr = invokeMethod(env, &jVal, STATIC, NULL,
        JC_URI, "create",
        "(Ljava/lang/String;)Ljava/net/URI;", jURIString);
// 有异常，则进行打印
if (jthr) {
    ret = printExceptionAndFree(env, jthr, PRINT_EXC_ALL,
        "hdfsBuilderConnect(%s)",
        hdfsBuilderToStr(bld, buf, sizeof(buf)));
    goto done;
}
... 
done:
    // Release unnecessary local references
	destroyLocalReference(env, jURI);
	free(cURI);

jthrowable getPendingExceptionAndClear(JNIEnv *env) {
    jthrowable jthr = (*env)->ExceptionOccurred(env);
    if (!jthr)
        return NULL;
    (*env)->ExceptionClear(env);
    return jthr;
}
```



在引发异常后，本地代码必须首先清除异常，然后再进行其他JNI调用。存在待处理异常时，可以安全调用的JNI函数有：

```c
ExceptionOccurred()
ExceptionDescribe()
ExceptionClear()
ExceptionCheck()
ReleaseStringChars()
ReleaseStringUTFChars()
ReleaseStringCritical()
Release<Type>ArrayElements()
ReleasePrimitiveArrayCritical()
DeleteLocalRef()
DeleteGlobalRef()
DeleteWeakGlobalRef()
MonitorExit()
PushLocalFrame()
PopLocalFrame()
DetachCurrentThread()
```



## JNI 类型和数据结构

### 基本类型

| Java类型 | 本机类型 | 描述       |
| :------- | :------- | :--------- |
| boolean  | jboolean | 无符号8位  |
| byte     | jbyte    | 有符号8位  |
| char     | jchar    | 无符号16位 |
| short    | jshort   | 有符号16位 |
| int      | jint     | 有符号32位 |
| long     | jlong    | 有符号64位 |
| float    | jfloat   | 32位       |
| double   | jdouble  | 64位       |
| void     | void     | 不适用     |

### 引用类型

JNI包括一些引用类型，对应不同类型的Java对象。JNI引用类型组织如下层次结构：

- ```
  jobject
  ```

  - `jclass`（`java.lang.Class`对象）
  - `jstring`（`java.lang.String`对象）
  - `jarray（数组）`
    - `jobjectArray`（对象数组）
    - `jbooleanArray`（`boolean`数组）
    - `jbyteArray`（`byte`数组）
    - `jcharArray`（`char`数组）
    - `jshortArray`（`short`数组）
    - `jintArray`（`int`数组）
    - `jlongArray`（`long`数组）
    - `jfloatArray`（`float`数组）
    - `jdoubleArray`（`double`数组）
  - `jthrowable`（`java.lang.Throwable`对象）

在C中，所有其他JNI引用类型被定义为与jobject相同。例如：

```c
typedef jobject jclass;
```

在C++中，JNI引入了一组虚拟类来强制子类型关系。例如：

```c++
class _jobject {};
class _jclass : public _jobject {};
// ...
typedef _jobject *jobject;
typedef _jclass *jclass;
```

### 字段和方法ID

方法和字段ID是常规的**C指针类型**：

```c
struct _jfieldID;              /* 不透明结构 */
typedef struct _jfieldID *jfieldID;   /* 字段ID */

struct _jmethodID;              /* 不透明结构 */
typedef struct _jmethodID *jmethodID; /* 方法ID */
```

### 值类型

`jvalue`联合类型用作参数数组中的元素类型。它声明如下：

```c
typedef union jvalue {
    jboolean z;
    jbyte    b;
    jchar    c;
    jshort   s;
    jint     i;
    jlong    j;
    jfloat   f;
    jdouble  d;
    jobject  l;
} jvalue;
```

### 类型签名

| 类型签名                  | Java类型   |
| :------------------------ | :--------- |
| `Z`                       | boolean    |
| `B`                       | byte       |
| `C`                       | char       |
| `S`                       | short      |
| `I`                       | int        |
| `J`                       | long       |
| `F`                       | float      |
| `D`                       | double     |
| `L` 完全限定类 `;`        | 完全限定类 |
| `[` 类型                  | 类型[]     |
| `(` 参数类型 `)` 返回类型 | 方法类型   |

例如，Java方法：

```java
long f (int n, String s, int[] arr);
```

具有以下类型签名：`(ILjava/lang/String;[I)J`

### 修改后的UTF-8字符串

JNI使用修改后的**UTF-8字符串**来表示各种字符串类型。修改后的UTF-8字符串与Java虚拟机使用的相同。

- 多字节字符的字节以大端（高字节在前）顺序存储在`class`文件中。

与标准UTF-8格式之间存在两个差异:

- 空字符`(char)0`使用两字节格式（0xC0 0x80）而不是一个字节格式（0x00）进行编码，确保了编码后的字符串中不会嵌入null字符。
- 仅使用标准**UTF-8的一个字节、两字节和三字节**格式
- 标准UTF-8**的四字节格式被使用自己的两次三字节**格式代替



## JNI 函数

### 常量

```c
#define JNI_FALSE 0
#define JNI_TRUE 1
// 用于JNI函数的一般返回值常量。
#define JNI_OK           0                 /* 成功 */
#define JNI_ERR          (-1)              /* 未知错误 */
#define JNI_EDETACHED    (-2)              /* 线程与VM分离 */
#define JNI_EVERSION     (-3)              /* JNI版本错误 */
#define JNI_ENOMEM       (-4)              /* 内存不足 */
#define JNI_EEXIST       (-5)              /* VM已创建 */
#define JNI_EINVAL       (-6)              /* 无效参数 */
```

### 版本信息

```c
// JNI_VERSION_1_8, JDK 17 对应 JNI_VERSION_10
jint GetVersion(JNIEnv *env);
```

### 类操作

```c
// 从原始类数据缓冲区 buf 加载类，返回后，buf 可以被丢弃。
// name: 要定义的类或接口的名称。字符串以修改后的UTF-8编码。此值可以为NULL。
// loader: 分配给定义类的类加载器。此值可以为NULL，表示“空类加载器”（或“引导类加载器”）。
jclass DefineClass(JNIEnv *env, const char *name, jobject loader, const jbyte *buf, jsize bufLen);

// name: 完全限定的类名（即包名，由“/”分隔，后跟类名）。如果名称以“[”（数组签名字符）开头，则返回一个数组类。字符串以修改后的UTF-8编码
// 不存在当前本地方法或其关联的类加载器时，使用ClassLoader.getSystemClassLoader，读取java.class.path中的类；
// 通过本地方法时，JNI_OnLoad和JNI_OnLoad_L，使用加载本地库的类的类加载器；JNI_OnUnload(_L)使用SystemClassLoader（因为加载时使用的类加载器可能不存在）
jclass FindClass(JNIEnv *env, const char *name);

// 如果clazz表示除Object类之外的任何类，则此函数返回表示clazz指定的类的超类的对象。
// 如果clazz指定Object类，或者clazz表示一个接口，则此函数返回NULL。
jclass GetSuperclass(JNIEnv *env, jclass clazz);

// 确定clazz1的对象是否可以安全地转换为clazz2。
// 如果以下任一情况为真，则返回JNI_TRUE：
// 	  第一个和第二个类参数引用相同的Java类。
//    第一个类是第二个类的子类。
//    第一个类将第二个类作为其接口之一。
jboolean IsAssignableFrom(JNIEnv *env, jclass clazz1, jclass clazz2);
```

### 模块操作

```c
// 返回类所属模块的java.lang.Module对象。如果类不在命名模块中，则返回类加载器的未命名模块。
// 如果类表示数组类型，则此函数返回元素类型的Module对象。如果类表示原始类型或void，则返回java.base模块的Module对象。
jobject GetModule(JNIEnv *env, jclass clazz);
```

### 线程操作

```c
// 测试对象是否为虚拟线程。
jboolean IsVirtualThread(JNIEnv *env, jobject obj);
```

### 异常

```c
// 导致抛出java.lang.Throwable对象。
jint Throw(JNIEnv *env, jthrowable obj);

// 使用由message指定的消息从指定类构造异常对象，并导致抛出该异常。
// clazz: java.lang.Throwable的子类，不得为NULL。
jint ThrowNew(JNIEnv *env, jclass clazz, const char *message);

// 确定是否正在抛出异常。异常保持被抛出状态，直到本地代码调用ExceptionClear()，或Java代码处理异常。
// 返回当前正在被抛出的异常对象，如果当前没有异常被抛出，则返回NULL。
jthrowable ExceptionOccurred(JNIEnv *env);

// 将异常和堆栈的回溯打印到系统错误报告通道，如stderr。调用此函数的副作用是清除挂起的异常。这是为调试提供的便利程序。
void ExceptionDescribe(JNIEnv *env);

// 清除当前正在被抛出的任何异常。如果当前没有异常被抛出，则此例程不起作用。
void ExceptionClear(JNIEnv *env);

// 引发致命错误，并且不希望虚拟机恢复。此函数不返回。
void FatalError(JNIEnv *env, const char *msg);

// 用于检查是否存在未决异常，而不创建异常对象的本地引用。
jboolean ExceptionCheck(JNIEnv *env);
```

### 全局和局部引用

```c
// 创建一个新的全局引用，指向obj参数引用的对象。 obj参数可以是全局引用或局部引用。 必须通过调用DeleteGlobalRef()显式处理全局引用。
// 如果可能返回NULL的情况包括：
//	obj引用为null
//	系统内存不足
//	obj是弱全局引用并且已被垃圾回收
jobject NewGlobalRef(JNIEnv *env, jobject obj);

// 删除由globalRef指向的全局引用。
void DeleteGlobalRef(JNIEnv *env, jobject globalRef);

// 删除由localRef指向的局部引用。
void DeleteLocalRef(JNIEnv *env, jobject localRef);

// 确保当前线程中至少可以创建给定数量的局部引用。 成功返回0；否则返回负数并抛出OutOfMemoryError。
jint EnsureLocalCapacity(JNIEnv *env, jint capacity);

// 创建一个新的局部引用帧，在其中至少可以创建给定数量的局部引用。 成功返回0，失败时返回负数并挂起OutOfMemoryError。
jint PushLocalFrame(JNIEnv *env, jint capacity);

// 弹出当前局部引用帧，释放所有局部引用，并为给定的result对象返回上一个局部引用帧中的局部引用。
// 如果不需要返回到先前帧的引用，则将result传递为NULL。
jobject PopLocalFrame(JNIEnv *env, jobject result);


```

### 弱全局引用

弱全局引用允许底层Java对象被垃圾回收。 弱全局引用可以在任何需要全局或局部引用的情况下使用。

`IsSameObject`可用于比较弱全局引用与非`NULL`局部或全局引用。 如果对象相同，则只要另一个引用未被删除，弱全局引用就不会变得等效于`NULL`。

- 不应依赖`IsSameObject(weakObj, NULL)`来确定将来的JNI函数调用中是否可以使用弱全局引用（作为非`NULL`引用），因为介入的垃圾回收可能会更改弱全局引用。
- 使用JNI函数`NewLocalRef`或`NewGlobalRef`获取对底层对象的（强）局部或全局引用。 如果对象已被释放，这些函数将返回`NULL`。 否则，新引用将防止底层对象被释放。

```c
// 创建一个新的弱全局引用。  可以使用IsSameObject来测试引用的对象是否已被释放。 如果obj引用为null，或者虚拟机内存不足，则返回NULL。 如果虚拟机内存不足，同时将抛出OutOfMemoryError。
jweak NewWeakGlobalRef(JNIEnv *env, jobject obj);

// 删除给定弱全局引用所需的VM资源。
void DeleteWeakGlobalRef(JNIEnv *env, jweak obj);
```



### 对象操作

```c
// 分配一个新的Java对象而不调用对象的任何构造函数。返回对象的引用。
// clazz参数不得引用数组类。
jobject AllocObject(JNIEnv *env, jclass clazz);

// 构造一个新的Java对象。方法ID指示要调用的构造方法。此ID必须通过使用GetMethodID()并将<init>作为方法名和void (V)作为返回类型来获取。
jobject NewObject(JNIEnv *env, jclass clazz, jmethodID methodID, ...);
jobject NewObjectA(JNIEnv *env, jclass clazz, jmethodID methodID, const jvalue *args);
jobject NewObjectV(JNIEnv *env, jclass clazz, jmethodID methodID, va_list args);

// 返回对象的类。
jclass GetObjectClass(JNIEnv *env, jobject obj);

// 返回由obj参数引用的对象的类型。参数obj可以是本地、全局或弱全局引用，也可以是NULL。
// JNIInvalidRefType    = 0
// JNILocalRefType      = 1
// JNIGlobalRefType     = 2
// JNIWeakGlobalRefType = 3
jobjectRefType GetObjectRefType(JNIEnv* env, jobject obj);

// 测试对象是否是类的实例。
jboolean IsInstanceOf(JNIEnv *env, jobject obj, jclass clazz);

// 测试两个引用是否指向相同的Java对象
jboolean IsSameObject(JNIEnv *env, jobject ref1, jobject ref2);
```

### 访问对象的字段

```c
// 返回类的实例（非静态）字段的字段ID。字段由其名称和签名指定。 导致一个未初始化的类被初始化。
// GetFieldID()不能用于获取数组的长度字段。请改用GetArrayLength()。
jfieldID GetFieldID(JNIEnv *env, jclass clazz, const char *name, const char *sig);

// 返回对象的实例（非静态）字段的值
// 如 jint GetIntField()
<NativeType> Get<type>Field(JNIEnv *env, jobject obj, jfieldID fieldID);

// 设置对象的实例（非静态）字段的值
// 如 SetIntField(JNIEnv *env, jobject obj, jfieldID fieldID, jint value)
void Set<type>Field(JNIEnv *env, jobject obj, jfieldID fieldID, <NativeType> value);
```

### 调用实例方法

```c
// 返回类或接口的实例（非静态）方法的方法ID，包括父类/接口中的方法。导致一个未初始化的类被初始化。
// 要获取构造函数的方法ID，请将<init>作为方法名称并将void（V）作为返回类型。
jmethodID GetMethodID(JNIEnv *env, jclass clazz, const char *name, const char *sig);

// 调用Java实例方法。它们在向调用的方法传递参数的机制上有所不同。
<NativeType> Call<type>Method(JNIEnv *env, jobject obj, jmethodID methodID, ...);
<NativeType> Call<type>MethodA(JNIEnv *env, jobject obj, jmethodID methodID, const jvalue *args);
<NativeType> Call<type>MethodV(JNIEnv *env, jobject obj, jmethodID methodID, va_list args);

// 多个 clazz 参数，调用指定类（自身类或者其超类）的方法，用于调用被子类覆盖的父类的方法。
<NativeType> CallNonvirtual<type>Method(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, ...);
<NativeType> CallNonvirtual<type>MethodA(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, const jvalue *args);
<NativeType> CallNonvirtual<type>MethodV(JNIEnv *env, jobject obj, jclass clazz, jmethodID methodID, va_list args);
```

### 访问静态字段

```c
// 返回类的静态字段的字段ID。
jfieldID GetStaticFieldID(JNIEnv *env, jclass clazz, const char *name, const char *sig);

// 返回对象的静态字段的值
<NativeType> GetStatic<type>Field(JNIEnv *env, jclass clazz, jfieldID fieldID);

// 设置对象的静态字段的值
void SetStatic<type>Field(JNIEnv *env, jclass clazz, jfieldID fieldID, <NativeType> value);
```

### 调用静态方法

```c
// 获取静态方法
jmethodID GetStaticMethodID(JNIEnv *env, jclass clazz, const char *name, const char *sig);

// 调用静态方法
<NativeType> CallStatic<type>Method(JNIEnv *env, jclass clazz, jmethodID methodID, ...);
<NativeType> CallStatic<type>MethodA(JNIEnv *env, jclass clazz, jmethodID methodID, jvalue *args);
<NativeType> CallStatic<type>MethodV(JNIEnv *env, jclass clazz, jmethodID methodID, va_list args);
```

### 字符串操作

这里的 unicode 表示 utf 16编码。

- [C/C++ 中的字符串常量的编码](https://en.cppreference.com/w/cpp/language/charset#Code_unit_and_literal_encoding)，默认是跟文件的存储编码相关的。

```c
// 从Unicode字符数组构造一个新的java.lang.String对象。
jstring NewString(JNIEnv *env, const jchar *unicodeChars, jsize len);

// 返回Java字符串的长度（Unicode字符数）。
jsize GetStringLength(JNIEnv *env, jstring string);

// 返回字符串的Unicode字符数组的指针。此指针有效直到调用ReleaseStringChars()。
// *isCopy 设置为 JNI_TRUE（如果创建副本）; 或者如果未创建副本，则设置为 JNI_FALSE。
const jchar * GetStringChars(JNIEnv *env, jstring string, jboolean *isCopy);

// 通知VM本地代码不再需要访问chars。
void ReleaseStringChars(JNIEnv *env, jstring string, const jchar *chars);

// 从修改后的UTF-8编码字符数组构造一个新的java.lang.String对象。
jstring NewStringUTF(JNIEnv *env, const char *bytes);

// 返回字符串的修改后的UTF-8表示的字节长度。
jsize GetStringUTFLength(JNIEnv *env, jstring string);

// 返回一个指向以修改后的UTF-8编码表示的字符串的字节数组的指针。该数组有效直到被ReleaseStringUTFChars()释放。
// *isCopy 设置为 JNI_TRUE（如果创建副本）; 或者如果未创建副本，则设置为 JNI_FALSE。
const char * GetStringUTFChars(JNIEnv *env, jstring string, jboolean *isCopy);

// 通知虚拟机本地代码不再需要访问utf。
void ReleaseStringUTFChars(JNIEnv *env, jstring string, const char *utf);

// 将从偏移量start开始的len个Unicode字符复制到给定的缓冲区buf中。
void GetStringRegion(JNIEnv *env, jstring str, jsize start, jsize len, jchar *buf);

// 将从偏移量start开始的len个Unicode字符转换为修改后的UTF-8编码，并将结果放入给定的缓冲区buf中。
// 生成的修改后的UTF-8编码字符数可能大于给定的len参数。可以使用GetStringUTFLength()来确定所需字符缓冲区的最大大小。
// 由于此规范不要求生成的字符串副本以NULL结尾，建议在使用此函数之前清除给定的字符缓冲区（例如"memset()"），以便安全地执行strlen()。
void GetStringUTFRegion(JNIEnv *env, jstring str, jsize start, jsize len, char *buf);

// 语义类似于现有的Get/ReleaseStringChars函数。如果可能，VM 将返回指向字符串元素的指针;否则，将创建一个副本。
// 但是这些功能的使用方式存在很大限制。在 Get/ReleaseStringCritical 调用包含的代码段中，本机代码不得发出任意 JNI 调用，也不得导致当前线程阻塞。
const jchar * GetStringCritical(JNIEnv *env, jstring string, jboolean *isCopy);
void ReleaseStringCritical(JNIEnv *env, jstring string, const jchar *carray);    
```

### 数组操作

```c
// 返回数组中的元素数量。
jsize GetArrayLength(JNIEnv *env, jarray array);

// 构造一个持有elementClass类中对象的新数组。所有元素最初设置为initialElement。
jobjectArray NewObjectArray(JNIEnv *env, jsize length, jclass elementClass, jobject initialElement);

// 返回Object数组的一个元素。用于对象数组，或者多维数组。
jobject GetObjectArrayElement(JNIEnv *env, jobjectArray array, jsize index);

// 设置Object数组的一个元素。
void SetObjectArrayElement(JNIEnv *env, jobjectArray array, jsize index, jobject value);

// 创建基本类型是数组，如  jintArray NewIntArray()
<ArrayType> New<PrimitiveType>Array(JNIEnv *env, jsize length);

// 返回原始数组的内容。结果在对应的Release<PrimitiveType>ArrayElements()函数被调用之前有效。
// 由于返回的数组可能是Java数组的副本，对返回的数组进行的更改不一定会反映在原始数组中，直到调用Release<PrimitiveType>ArrayElements()为止。
<NativeType> *Get<PrimitiveType>ArrayElements(JNIEnv *env, <ArrayType> array, jboolean *isCopy);

// 通知虚拟机本机代码不再需要访问elems.
// mode参数(默认用0即可）提供有关如何释放数组缓冲区的信息。如果elems不是数组中元素的副本，则mode不起作用。否则，mode具有以下影响，如下表所示:
// 模式		   操作
// 0			复制回内容并释放elems缓冲区
// JNI_COMMIT	复制回内容但不释放elems缓冲区
// JNI_ABORT	释放缓冲区而不复制回可能的更改
void Release<PrimitiveType>ArrayElements(JNIEnv *env, <ArrayType> array, NativeType *elems, jint mode);

// 将原始数组的区域复制到缓冲区中。
void Get<PrimitiveType>ArrayRegion(JNIEnv *env, <ArrayType> array, jsize start, jsize len, <NativeType> *buf);
// 从缓冲区中将原始数组的区域复制回去。
void Set<PrimitiveType>ArrayRegion(JNIEnv *env, ArrayType array, jsize start, jsize len, const NativeType *buf);
```

### 注册本地方法

```c
// 使用clazz参数指定的类注册本机方法，即将 Java native 方法映射为 C 中的方法。
// JNINativeMethod结构的name和signature字段是指向修改后的UTF-8字符串的指针。 nMethods参数指定数组中的本机方法数量。 
// typedef struct {
//    char *name;
//    char *signature;
//    void *fnPtr;
// } JNINativeMethod;
// 函数指针名义上必须具有以下签名：
// ReturnType (*fnPtr)(JNIEnv *env, jobject objectOrClass, ...);
ReturnType (*fnPtr)(JNIEnv *env, jobject objectOrClass, ...);
jint RegisterNatives(JNIEnv *env, jclass clazz, const JNINativeMethod *methods, jint nMethods);

// 取消注册类的本机方法。 类返回到链接或注册其本机方法函数之前的状态。
jint UnregisterNatives(JNIEnv *env, jclass clazz);
```

### 监视器操作

每个Java对象都有一个与之关联的监视器。

- 如果当前线程已经拥有与`obj`关联的监视器，则会增加监视器中的计数器，指示该线程进入监视器的次数
- 如果与`obj`关联的监视器没有被任何线程拥有，则当前线程将成为监视器的所有者，将该监视器的进入计数设置为1。
- 如果另一个线程已经拥有与`obj`关联的监视器，则当前线程将等待直到监视器被释放，然后再次尝试获取所有权。

通过`MonitorEnter` JNI函数调用进入的监视器不能使用`monitorexit` Java虚拟机指令或同步方法返回退出。`MonitorEnter` JNI函数调用和`monitorenter` Java虚拟机指令可能会竞争进入与同一对象关联的监视器。

为避免死锁，通过`MonitorEnter` JNI函数调用进入的监视器必须使用`MonitorExit` JNI调用退出，除非使用`DetachCurrentThread`调用隐式释放JNI监视器。

本机代码不能使用`MonitorExit`退出通过同步方法或`monitorenter` Java虚拟机指令进入的监视器。

```c
// 进入与obj引用的底层Java对象关联的监视器。
jint MonitorEnter(JNIEnv *env, jobject obj);

// 当前线程必须是与obj引用的底层Java对象关联的监视器的所有者。线程减少指示它进入此监视器的次数。如果计数器的值变为零，则当前线程释放监视器。
jint MonitorExit(JNIEnv *env, jobject obj);
```

### NIO 支持

```c
// 分配并返回一个直接引用内存地址为address、长度为capacity字节的java.nio.ByteBuffer。返回的缓冲区的字节顺序始终为大端序。
// capacity 不能为负数或大于Integer.MAX_VALUE
jobject NewDirectByteBuffer(JNIEnv* env, void* address, jlong capacity);

// 获取并返回给定直接java.nio.Buffer引用的内存区域的起始地址。允许本机代码访问与Java代码通过缓冲区对象访问的相同内存区域。
void* GetDirectBufferAddress(JNIEnv* env, jobject buf);

// 获取并返回给定直接java.nio.Buffer引用的内存区域的容量。容量是内存区域包含的元素数量。
jlong GetDirectBufferCapacity(JNIEnv* env, jobject buf);
```

### 反射支持

JNI提供了一组在JNI中使用的字段和方法ID与Java核心**反射API中使用的字段和方法对象**之间进行转换的函数。

```c
// 将java.lang.reflect.Method或java.lang.reflect.Constructor对象转换为方法ID。
// method：一个java.lang.reflect.Method或java.lang.reflect.Constructor对象，不得为NULL。
jmethodID FromReflectedMethod(JNIEnv *env, jobject method);

// 将java.lang.reflect.Field转换为字段ID。
jfieldID FromReflectedField(JNIEnv *env, jobject field);

// 将从cls派生的方法ID转换为java.lang.reflect.Method或java.lang.reflect.Constructor对象。
// 设置为JNI_TRUE，如果方法ID引用静态字段，否则设置为JNI_FALSE
jobject ToReflectedMethod(JNIEnv *env, jclass cls, jmethodID methodID, jboolean isStatic);

// 将从cls派生的字段ID转换为一个java.lang.reflect.Field对象。
jobject ToReflectedField(JNIEnv *env, jclass cls, jfieldID fieldID, jboolean isStatic);
```

### VM 接口

```c
// 返回与当前线程关联的Java VM接口（在调用API中使用）。结果放置在第二个参数vm指向的位置。
// 成功返回"0"；失败返回负值。
jint GetJavaVM(JNIEnv *env, JavaVM **vm);
```



## 调用 API

调用API允许软件供应商将Java虚拟机加载到任意本机应用程序中。

### 概述

示例C++代码创建了一个Java虚拟机，并调用了一个名为`Main.test`的静态方法

```c++
#include <jni.h>       /* 定义所有内容的位置 */
...
JavaVM *jvm;       /* 表示Java虚拟机 */
JNIEnv *env;       /* 指向本机方法接口的指针 */
JavaVMInitArgs vm_args; /* JDK/JRE 19 VM初始化参数 */
JavaVMOption* options = new JavaVMOption[1];
options[0].optionString = "-Djava.class.path=/usr/lib/java";
vm_args.version = JNI_VERSION_19;
vm_args.nOptions = 1;
vm_args.options = options;
vm_args.ignoreUnrecognized = false;
/* 加载和初始化Java虚拟机，返回一个JNI接口指针
 * 在env中 */
JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
delete options;
/* 使用JNI调用Main.test方法 */
jclass cls = env->FindClass("Main");
jmethodID mid = env->GetStaticMethodID(cls, "test", "(I)V");
env->CallStaticVoidMethod(cls, mid, 100);
/* 完成。 */
jvm->DestroyJavaVM();
```

#### 创建虚拟机

`JNI_CreateJavaVM()`函数加载并初始化Java虚拟机，并返回一个JNI接口指针。

- 调用`JNI_CreateJavaVM()`的线程被视为*主线程*，并附加到Java虚拟机。

#### 附加到虚拟机

JNI接口指针（`JNIEnv`）仅在当前线程中有效。如果另一个线程需要访问Java虚拟机，先调用`AttachCurrentThread()`将自己附加到虚拟机并获取JNI接口指针。

- 附加的线程应该有足够的堆栈空间来执行合理数量的工作。用pthread时，堆栈大小可以在`pthread_create`的`pthread_attr_t`参数中指定。

#### 从虚拟机分离

附加到虚拟机的本机线程在终止之前必须调用`DetachCurrentThread()`将自己分离。如果调用堆栈上有Java方法，则线程无法分离自己。

#### 终止虚拟机

`DestroyJavaVM()`函数终止Java虚拟机。

- 待直到**没有非守护线程**在执行，然后才实际终止虚拟机；非守护线程包括**Java线程和附加的本机线程**。

### 库和版本管理

> **相同的 JNI 本机库不能加载到多个类加载器中。**

当使用`System.loadLibrary`将本机库加载到两个类加载器中时，会抛出`UnsatisfiedLinkError`。这种方法的好处包括：

- 基于类加载器的名称空间分离在本机库中得以保留。本机库不能轻松混合来自不同类加载器的类。
- 此外，当相应的类加载器被垃圾回收时，本机库可以被卸载。

#### 静态链接库支持



#### 库生命周期函数挂钩

为了便于版本控制和资源管理，JNI库可以定义***加载***和***卸载***函数挂钩。这些函数的命名取决于库是动态链接还是静态链接。

#### JNI_OnLoad

```c
// reserved：未使用的指针。返回所需的JNI_VERSION常量
jint JNI_OnLoad(JavaVM *vm, void *reserved);
```

`JNI_OnLoad`必须返回至少定义JNI API 版本的常量：

- 如果本机库不导出`JNI_OnLoad`函数，则VM假定该库仅需要JNI版本`JNI_VERSION_1_1`。

#### JNI_OnUnload

```c
//当包含本机库的类加载器被垃圾回收时，VM会调用JNI_OnUnload。
void JNI_OnUnload(JavaVM *vm, void *reserved);
```

此函数可用于执行清理操作。由于此函数在未知上下文中调用（例如从终结器调用），程序员在使用Java VM服务时应保守，并避免任意的Java回调。

#### JNI_OnLoad_L

```c
jint JNI_Onload_<L>(JavaVM *vm, void *reserved);
```

如果一个名为'L'的库是静态链接的，那么在第一次调用`System.loadLibrary("L")`或等效的API时，将调用一个具有与`JNI_OnLoad`函数指定的相同参数和期望返回值的`JNI_OnLoad_L`函数。

- 返回本地库所需的JNI版本，此版本必须是`JNI_VERSION_1_8`或更高版本。

#### JNI_OnUnload_L

```c
void JNI_OnUnload_<L>(JavaVM *vm, void *reserved);
```

当包含静态链接本地库'L'的类加载器被垃圾回收时，如果导出了`JNI_OnUnload_L`函数，则VM将调用该库的`JNI_OnUnload_L`函数。

### 调用API函数

`JavaVM`类型是指向调用API函数表的指针。

```c
typedef const struct JNIInvokeInterface *JavaVM;

const struct JNIInvokeInterface ... = {
    NULL,
    NULL,
    NULL,

    DestroyJavaVM,
    AttachCurrentThread,
    DetachCurrentThread,

    GetEnv,

    AttachCurrentThreadAsDaemon
};
```

#### JNI_GetDefaultJavaVMInitArgs

```c
// vm_args: JavaVMInitArgs结构的指针，不能为NULL
// 如果支持请求的版本，则返回JNI_OK；如果不支持请求的版本，则返回JNI错误代码（负数）
jint JNI_GetDefaultJavaVMInitArgs(void *vm_args);
```

返回Java VM的默认配置。在调用此函数之前，本机代码必须将`vm_args->version`字段设置为它期望VM支持的JNI版本。

- 此函数返回后，vm_args->version将设置为VM支持的实际JNI版本。  

#### JNI_GetCreatedJavaVMs

```c
// vmBuf：指向将放置VM结构的缓冲区的指针，不能为NULL
// 成功时返回JNI_OK；失败时返回适当的JNI错误代码（负数）。
jint JNI_GetCreatedJavaVMs(JavaVM **vmBuf, jsize bufLen, jsize *nVMs);
```

返回已创建的所有Java VM。将VM指针按创建顺序写入缓冲区`vmBuf`。最多将写入`bufLen`个条目。已创建的VM总数将在`\*nVMs`中返回。

- 不支持在单个进程中创建多个VM。

#### JNI_CreateJavaVM

```c
// p_vm：指向将放置结果VM结构的位置的指针。不能为NULL。
// p_env：指向将放置主线程的JNI接口指针的位置的指针。不能为NULL。
// vm_args：Java VM初始化参数。不能为NULL。
jint JNI_CreateJavaVM(JavaVM **p_vm, void **p_env, void *vm_args);
```

加载并初始化Java VM。当前线程将附加到Java VM并成为主线程。将`p_env`参数设置为主线程的JNI接口指针。

- 不支持在单个进程中创建多个VM。

`JavaVMInitArgs`结构如下：

```c
typedef struct JavaVMInitArgs {
    jint version;
    jint nOptions;  // options 数组的大小
    JavaVMOption *options;
    jboolean ignoreUnrecognized; //为JNI_TRUE时，忽略所有以"-X"或"_"开头的未识别选项字符串
} JavaVMInitArgs;
// options字段是以下类型的数组：
typedef struct JavaVMOption {
    char *optionString;  /* 以默认平台编码的字符串形式表示的选项 */
    void *extraInfo;
} JavaVMOption;
```

示例创建JVM：

```c
JavaVMInitArgs vm_args;
JavaVMOption options[3];
// 创建JVM，涉及到第三方jar包时，通过**"-Djava.class.path=<path_to_my_java_class>" 指定，无论jar包还是class文件**，不能是目录；
options[0].optionString = "-Djava.class.path=c:\myclasses"; /* 用户类 */
options[1].optionString = "-Djava.library.path=c:\mylibs";  /* 设置本地库路径 */
options[2].optionString = "-verbose:jni";                   /* 打印JNI相关消息 */

vm_args.version = JNI_VERSION_1_2;
vm_args.options = options;
vm_args.nOptions = 3;
vm_args.ignoreUnrecognized = TRUE;

/* 请注意，在JDK/JRE中，不再需要调用
 * JNI_GetDefaultJavaVMInitArgs。
 */
res = JNI_CreateJavaVM(&vm, (void **)&env, &vm_args);
if (res < 0) ...
```

#### DestroyJavaVM

任何线程，无论是否已附加，都可以调用此函数。如果当前线程未附加，则首先将其附加。如果当前线程已附加，则如果其调用堆栈上有任何Java方法，则出现错误。见[终止虚拟机](#终止虚拟机)。

#### AttachCurrentThread

```c
jint AttachCurrentThread(JavaVM *vm, void **p_env, void *thr_args);
```

将当前线程附加到Java虚拟机作为*非守护*线程。见[附加到虚拟机](#附加到虚拟机)。

- 已经附加的线程的守护状态通过调用此方法不会改变。
- 当线程附加到VM时，上下文类加载器是引导加载器。

`thr_args`：可以为NULL或指向`JavaVMAttachArgs`结构的指针以指定附加信息：

```c
typedef struct JavaVMAttachArgs {
    jint version;
    char *name;    /* 线程的名称作为修改后的UTF-8字符串，或为NULL */
    jobject group; /* ThreadGroup对象的全局引用，或为NULL */
} JavaVMAttachArgs
```

#### AttachCurrentThreadAsDaemon

将当前线程附加到Java虚拟机作为*守护*线程。

- 已经附加的线程的守护状态通过调用此方法不会改变。
- 当线程附加到VM时，上下文类加载器是引导加载器。

#### DetachCurrentThread

将当前线程从Java虚拟机中分离。如果调用堆栈上有Java方法，则线程无法分离自身。

- 主线程可以从VM中分离；尝试分离未附加的线程是一个空操作。
- 当在一个线程里面调用**AttachCurrentThread**后，如果不需要用的时候一定要**DetachCurrentThread**，否则线程无法正常退出。

#### GetEnv

```c
// 如果当前线程未附加到VM，则将*env设置为NULL，并返回JNI_EDETACHED。如果指定的版本不受支持，则将*env设置为NULL，并返回JNI_EVERSION。否则，将*env设置为适当的接口，并返回JNI_OK。
jint GetEnv(JavaVM *vm, void **p_env, jint version);
```

