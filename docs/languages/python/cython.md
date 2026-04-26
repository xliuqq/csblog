# Cython

> **Cython** is an **optimising static compiler** for both the **[Python](http://www.python.org/about/)** programming language and the extended Cython programming language (based on **Pyrex**). It makes writing C extensions for Python as easy as Python itself.

*Cython*定义单独的一门语言，专门用来写**在Python里面import用的扩展库**。本身是个python 包。

## 编译

Cython 会先把 .PXD,    .PY,   .PYW,  .PYX 文件转换成 .C 中间代码， 再编译成 windows中.pyd和linux中.so文件。

**cython**

- 将`py`或`pyx`文件编译成C/C++文件；

**cythonize**

```shell
# 生成.c 和.so 文件
cythonize -a -i yourmod.pyx
```

- 将`py`或`pyx`文件编译成C/C++文件，将C/C++文件编译成扩展模块，可以直接从python中导入。
- 接收多个源文件和glob模式如`**/*.pyx`

### pyx文件

Cython源码文件

### pxd文件

Cython头文件。

### pyd文件

.pyd 文件是由非 Python的编程语言编写 (或直接把 .py 文件转换成 .c 中间文件) 编译生成的 Python 扩展模块，是类似 .so, .dll 动态链接库的一种 Python 文件。

Cython 可将个人基于 Python 语言编写的 Python 模块编译成具有 C 语言特性的 .pyd 文件。



## 

