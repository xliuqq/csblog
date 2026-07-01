# Python

## 解释器

### CPython

**官方**版本的解释器：Cpython，用 C 语言开发的，运行 python，就是启动 CPython 解释器，CPython 是使用最广的Python解释器。

### IPython

IPython是基于 CPython 之上的一个**交互式**解释器。

### PyPy

PyPy 是用 Python 实现的解释器，PyPy 采用 JIT 技术，对 Python 代码进行动态编译。

### Jython

Jython是运行在Java平台上的Python解释器，可以直接把Python代码编译成Java字节码执行。


## 字符串

### format

```python
a = "I'm %s. I'm %d year old" % ('Vamei', 99)
# or
"I'm %(name)s. I'm %(age)d year old" % {'name':'Vamei', 'age':99}
# or
"I'm {name}. I'm {age} year old".format(name='Vamei', age=99)
# or
name='Vamei', age=99
f"I'm {name}. I'm {age} year old"
```

| 格式      |         |  |              |
| ----------------------------- | ---------------- | -------------------- | ---------------------------------- |
| %s  字符串 (采用str()的显示)  | %d  十进制整数   | %e  指数 (基底写为e) | %g  指数(e)或浮点数 (根据显示长度) |
| %r  字符串 (采用repr()的显示) | %i  十进制整数   | %E  指数 (基底写为E) | %G  指数(E)或浮点数 (根据显示长度) |
| %c  单个字符     | %o  八进制整数   | %f  浮点数   | %%  字符"%"                        |
| %b  二进制整数       | %x  十六进制整数 | %F  浮点数，与上相同 |                |

### string 开头r b u f

```python
# bytes字节符，打印以b开头
b'input\n'
# 非转义原生字符，经处理'\n'变成了'\\'和'n'。也就是\n表示的是两个字符，而不是换行。
r'input\n'               
# unicode编码字符，python3默认字符串编码方式。
u'input\n'          
# 变量替换
f"I'm {name}. I'm {age} year old" 
```

如果要将现有字符串转换为原始字符串，那么我们可以像下面那样重新分配 

```python
s1 = "welcome\tto\tPython" 
raw_s1 = "%r" % s1 
print(raw_s1) # 会打印 welcome\tto\tPython
```

### 转为字符串方法

- str() :  字符串的形式
- repr() : 作用在字符串上时会多一层引号



## 子进程(TODO)

os.popen



## -m

python -m run.py

作用是：把 run.py 文件当做模块启动

**直接启动是把run.py文件，所在的目录放到了sys.path属性中。模块启动是把你输入命令的目录（也就是当前路径），放到了sys.path属性**



## Psutil

在Python中获取系统信息的模块，包括：

- CPU信息，逻辑核数、物理核数，cpu_times（用户/系统/空闲时间）；
- top；
- 物理内存和交换区信息；
- 磁盘信息，磁盘分区、使用率和IO；
- 网络接口和网络连接信息；
- 进程的详细信息；



## 类

### super

https://mozillazg.com/2016/12/python-super-is-not-as-simple-as-you-thought.html

- Python 3 可以使用直接使用 **super().xxx 代替 super(Class, self).xxx**

- super 包含了两个非常重要的信息: 一个 **MRO(Method Resolution Order) 列表**以及 MRO 中的一个类。当以如下方式调用 super 时: super(a_type, obj)  MRO 列表指的是 type(obj) 的 MRO 列表, MRO 中的那个类就是 a_type , 同时 isinstance(obj, a_type) == True 。

- 举个栗子, 有个 MRO 列表:

  [A, B, C, D, E, object]

  下面的调用:

  super(C, A).foo()

  super 只会从 C 之后查找，即: 只会在 D 或 E 或 object 中查找 foo 方法。



## AtExit

python atexit 模块定义了一个 **register 函数，用于在 python 解释器中注册一个退出函数**，这个函数在解释器正常终止时自动执行,一般用来做一些资源清理的操作。 atexit 按注册的相反顺序执行这些函数; 例如注册A、B、C，在解释器终止时按顺序C，B，A运行。

Note：如果程序是非正常crash，或者通过os._exit()退出，注册的退出函数将不会被调用。

### 使用方式

```python
from atexit import register
```

装饰器 @register 或者register调用

