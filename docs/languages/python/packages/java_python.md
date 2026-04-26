# Java/Python互相调用

## [JPype](https://github.com/jpype-project/jpype)

> JPype is cross language bridge to allow Python programs full access to Java class libraries.



## [PyJnius](https://github.com/kivy/pyjnius)

A Python module to access Java classes as Python classes using the Java Native Interface (JNI)

### 入门

```shell
pip install pyjnius
```

```python
>>> from jnius import autoclass
>>> autoclass('java.lang.System').out.println('Hello world')
Hello world

>>> Stack = autoclass('java.util.Stack')
>>> stack = Stack()
>>> stack.push('hello')
>>> stack.push('world')
>>> print stack.pop()
world
>>> print stack.pop()
hello
```

## [Py4j](https://www.py4j.org/)

> Py4J enables Python programs to dynamically access arbitrary Java objects, also enables Java programs to call back Python objects.

Python for Java，通过 **socket** 进行消息发送，使得 Python 中可以调用 Java代码 / Java 也可以回调Python对象。

### 入门

Java 端启动服务：

```java
GatewayServer gatewayServer = new GatewayServer(new EntryPoint(), 25555);
gatewayServer.start();
System.out.println("Gateway Server Started");
```

Python端：

```python
from py4j.java_gateway import JavaGateway, GatewayParameters

gateway = JavaGateway(gateway_parameters=GatewayParameters(port=25555))
# 获取String类型
JString = gateway.jvm.java.lang.String

# 转换成Python中的字符串
a = JString("213")
```

TODO: Java 回调 Python



# 序列化

## `net.razorvine.pickle`(https://github.com/irmen/pickle)

> Java and .NET library for Python's pickle serialization protocol

Apache Spark采用该库将Java对象序列化成Pickle格式，然后Python进行读取。



Java 读写 pickle （Python序列化格式）文件。

Pickle protocol version support: 

- reading: 0,1,2,3,4,5;
- writing: 2. 



### Python to Java (unpickling)

| PYTHON                                     | JAVA                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| None                                       | null                                                         |
| bool                                       | boolean                                                      |
| int                                        | int                                                          |
| long                                       | long or BigInteger (depending on size)                       |
| string                                     | String                                                       |
| unicode                                    | String                                                       |
| complex                                    | net.razorvine.pickle.objects.ComplexNumber                   |
| datetime.date                              | java.util.Calendar                                           |
| datetime.datetime                          | java.util.Calendar                                           |
| datetime.time                              | net.razorvine.pickle.objects.Time                            |
| datetime.timedelta                         | net.razorvine.pickle.objects.TimeDelta                       |
| float                                      | double (float isn't used)                                    |
| array.array                                | array of appropriate primitive type (char, int, short, long, float, double) |
| list                                       | java.util.List                                               |
| tuple                                      | Object[]                                                     |
| set                                        | java.util.Set                                                |
| dict                                       | java.util.Map                                                |
| bytes                                      | byte[]                                                       |
| bytearray                                  | byte[]                                                       |
| decimal                                    | BigDecimal                                                   |
| custom class                               | Map<String, Object> (dict with class attributes including its name in "**class**") |
| Pyro4.core.URI                             | net.razorvine.pyro.PyroURI                                   |
| Pyro4.core.Proxy                           | net.razorvine.pyro.PyroProxy                                 |
| Pyro4.errors.*                             | net.razorvine.pyro.PyroException                             |
| Pyro4.utils.flame.FlameBuiltin             | net.razorvine.pyro.FlameBuiltin                              |
| Pyro4.utils.flame.FlameModule              | net.razorvine.pyro.FlameModule                               |
| Pyro4.utils.flame.RemoteInteractiveConsole | net.razorvine.pyro.FlameRemoteConsole                        |

### Java to Python (pickling)

| JAVA                         | PYTHON                                                       |
| ---------------------------- | ------------------------------------------------------------ |
| null                         | None                                                         |
| boolean                      | bool                                                         |
| byte                         | int                                                          |
| char                         | str/unicode (length 1)                                       |
| String                       | str/unicode                                                  |
| double                       | float                                                        |
| float                        | float                                                        |
| int                          | int                                                          |
| short                        | int                                                          |
| BigDecimal                   | decimal                                                      |
| BigInteger                   | long                                                         |
| any array                    | array if elements are primitive type (else tuple)            |
| Object[]                     | tuple (cannot contain self-references)                       |
| byte[]                       | bytearray                                                    |
| java.util.Date               | datetime.datetime                                            |
| java.util.Calendar           | datetime.datetime                                            |
| java.sql.Date                | datetime.date                                                |
| java.sql.Time                | datetime.time                                                |
| java.sql.Timestamp           | datetime.datetime                                            |
| Enum                         | the enum value as string                                     |
| java.util.Set                | set                                                          |
| Map, Hashtable               | dict                                                         |
| Vector, Collection           | list                                                         |
| Serializable                 | treated as a JavaBean, see below.                            |
| JavaBean                     | dict of the bean's public properties + `__class__` for the bean's type. |
| net.razorvine.pyro.PyroURI   | Pyro4.core.URI                                               |
| net.razorvine.pyro.PyroProxy | cannot be pickled.                                           |

