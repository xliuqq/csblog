# Call Python

## GIL

在使用CPython解释器时，要注意GIL（全局解释锁）的工作原理以及对性能的影响。GIL保证在任意时刻只有一个线程在解释器中运行。在多线程环境中，python解释器工作原理如下： Plain Text

```text
1. 设置GIL
2. 切换到一个线程去运行
3. 运行：
    a. 指定数量的字节码指令，或者
    b. 线程主动让出控制（可以调用time.sleep(0)）
4. 把线程设置为睡眠状态
5. 解锁GIL
6. 再次重复以上所有步骤
```

GIL是一个历史遗留问题，导致**CPython多线程不能利用多个CPU内核的计算能力**。为了利用多核，通常使用多进程的方法，或是通过Python调用C代码，由C来实现多线程。

注意，当**在C/C++创建的线程中调用Python时，GIL需要通过函数PyGILState_Ensure()和PyGILState_Release(）手动获取、释放**。 

```C++
PyGILState_STATE gstate;
gstate = PyGILState_Ensure();

/* Perform Python actions here. */
result = CallSomeFunction();
/* evaluate result or handle exception */

/* Release the thread. No Python API allowed beyond this point. */
PyGILState_Release(gstate);
```

## 函数

### 加载python模块

```c++
PyRun_SimpleString("import sys"); //导入系统模块
PyRun_SimpleString("sys.path.append('./')"); //指定pytest.py所在的目录
PyObject* fname = PyUnicode_FromString("hello");
PyObject* pModule = PyImport_Import(fname);
if (pModule == NULL)
{
    PyErr_Print();
    std::exit(1);
}
```

### 调用模块的函数

#### 加载模块内的函数

```c++
PyObject *pfunc, *args, *results;
// pModule 是上一步load好的Python模块
Pfunc= PyObject_GetAttrString(pModule, "func");
// 设置调用func时的输入变量，这里假设为12345
args = Py_BuildValue("(i)",12345); 
// 执行func(12345)，并将结果返回给results
results= PyObject_CallObject(Pfunc, args); 
```

#### 加载模块内的类

```c++
PyObject *pClass, *pDict,*pInstance, *class_args, *results;
// 拿到pModule里的所有类和函数定义
pDict = PyModule_GetDict(pModule); 
// 找到名为Executor的类
pClass=PyDict_GetItemString(pDict,"Executor"); 
// 设置类初始化需要的参数
class_args = Py_BuildValue("(s)","./config.txt"); 
// 初始化Executor，建立实例pInstance
pInstance=PyInstance_New(pClass, class_args, NULL ); 
// 执行pInstance.func(12345)
results=PyObject_CallMethod(pInstance,"func","(i)",12345); 
```

#### 参数传递

参数格式见：[Parsing arguments and building values](https://link.zhihu.com/?target=https%3A//docs.python.org/2/c-api/arg.html)

```c++
PyObject* args = Py_BuildValue("(ifs)", 100, 3.14, "hello");
// 如果输入参数是另一个Python函数的输出结果PyObject* results (o)
PyObject* args = Py_BuildValue("(o)", results);
```

#### 解析返回值

同C++传递参数到Python类似，调用解析函数

单个返回值：PyArg_Parse()

多返回值：PyArg_ParseTuple()

具体例子：[1.7 Format Strings for PyArg_ParseTuple()](https://link.zhihu.com/?target=https%3A//docs.python.org/2.0/ext/parseTuple.html)

### 释放引用

> 在使用 `Py_DECREF` 函数时，需要确保对象的引用计数大于 0，否则会导致内存错误。通常情况下，在获取 Python 对象后，会自动增加其引用计数，因此需要在使用完后及时减少引用计数。

```c++
Py_DECREF(args);
```



## Demo

```c++
#include <Python.h>
int main(int argc, char *argv[]) {
    Py_Initialize();
    PyRun_SimpleString("print('hello world')\n");
    Py_Finalize();
    return 0;
}
// g++ -o main main.cpp -I/home/conda/envs/HXAI/include/python3.6m -L/home/conda/envs/HXAI/lib/ -lpython3.6m
// 输出 hello world
```



## 问题

### ImportError: **/python3.6/lib-dynload/_struct.cpython-36m-x86_64-linux-gnu.so: undefined symbol: PyDict_Size

conda装的python中其lib-dynload没有link python的动态库，通过`ldd -r _struct.cpython-36m-x86_64-linux-gnu.so`，会发现很多的unfined symbol。

因此需要提前加载python的动态库，使用环境变量`LD_PRELOAD=/home/conda/envs/HXAI/lib/libpython3.6m.so`实现。