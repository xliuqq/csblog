# Python性能分析工具

## profile

 纯python语言实现，返回**函数整体损耗**。



## cProfile

cProfile是标准版Python解释器默认的性能分析器。

cProfile是一种确定性分析器，**只测量CPU时间，不关心内存**。

同profile，部分实现native化，返回**函数整体损耗**。

```shell
python -m cProfile del.py
```



## Line Profile

**代码行级别的CPU时间监控**

`pip install line_profiler`



## Memory Profiler

获取进程的内存申请（而不是实际使用）的数量，支持行级别的监控；





## [Scalene](https://github.com/plasma-umass/scalene)：内存 / CPU/GPU 分析器

> Scalene: a high-performance CPU, GPU and memory profiler for Python.

### 功能

- 行和函数：报告有关整个函数和每个独立代码行的信息；
- 线程：支持 Python 线程；
- 多进程处理：支持使用 multiprocessing 库；
- Python 与 C 的时间：Scalene 用在 Python 与本机代码（例如库）上的时间；
- 系统时间：区分系统时间（例如，休眠或执行 I / O 操作）；
- GPU：报告在英伟达 GPU 上使用的时间（如果有）；
- 复制量：报告每秒要复制的数据量；
- 泄漏检测：自动查明可能造成内存泄漏的线路。

### 使用

安装：

```
pip install scalene
```

Scalene 的使用非常简单：

```shell
scalene <yourapp.py>
```

也可以使用魔术命令在 Jupyter notebook 中使用它：

```python
%load_ext scalene
```

输出结果如图：

![img](https://raw.githubusercontent.com/plasma-umass/scalene/master/docs/scalene-gui-example-full.png)
