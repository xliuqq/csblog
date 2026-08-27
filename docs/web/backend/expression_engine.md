# 表达式引擎

## [Aviator](https://github.com/killme2008/aviatorscript)

专用脚本 / 表达式引擎，自研语法，ASM 生成字节码，面向业务规则、公式计算。

- 支持自定义变量解析、自定义函数；

```java
// Aviator
Expression exp = AviatorEvaluator.compile("a>100 && contains(list, 'abc')");
Object res = exp.execute(AviatorEvaluator.newEnv("a",200,"list",Arrays.asList("abc")));
```



## [JEXL(Apache Commons)](https://commons.apache.org/proper/commons-jexl/)

Java Expression Language (JEXL) 是一个表达式语言引擎，可以用来校验数据。解释执行 EL 脚本，AST 解释，**不生成字节码**

### 原理

Engine：创建expression；

Context：包含变量；

Expression：语法表达式。

### 语法

#### 语言元素

##### 注释

支持`##`，`//`单行注释，以及`/* */`多行注释。

##### 变量名

以`a-z`, `A-Z`, `_` 或`$`开头，后续支持 `0-9`, `a-z`, `A-Z`, `_` 或`$`。大小写敏感。

#### 常量

Int, Float, Long, Double, BigInteger, BigDecimal, Octal, Hex, Real, String, Boolean, Null；

Multiline format literals：类似`Hello world`；

正则匹配常量：类似`~/ABC.*/`；

Array常量：`[ 1, 2, "three",...]`， `[]`表示空数组；

List常量：`[ 1, 2, "three",...]`， `[...]`表示空列表；

Set常量：`{ "one" , 2, "more"}`；

Map常量：`{ "one" : 1, "two" : 2, "three" : 3, "more": "many more" }；`

Range变量：`1 .. 42`，可以用作for循环变量中。

#### Operators（TODO）

#### 函数

`empty、size、new、ns:function`

`JexlEngine`允许注册对象或者类被用作函数名空间，允许类似`math:cosinus(23.0)`的表达式。

可以注册没有名空间的函数，即作为顶层的函数。

#### 访问（Access）

##### 方法

同样的方式调用对象的方法

##### JavaBean属性

```java
// 通过getter/setters进行属性的读写或者public属性，对于getBar()方法
foo['bar']; foo.bar;

// 针对Foo.getAttribute(String index) ，可以通过以下调用
x.attribute['name']; x.attribute.name;

// 针对Object get(String name) ，可以通过以下调用
foo['bar']; foo.bar;
```

##### 数组/列表

```java
// array 或者 list 获取元素
arr[0]
arr.0
```

##### 映射

```java
// map 获取元素（key不一定是string，只要是java object即可）
map[0]; map['name']; map[var];
```



#### 条件语句

##### if

```java
if ((x * 2) == 5) { y = 1; } else { y = 2; }
```

##### for

```java
// item是context全局变量
for (item : list) { x = x + item; }
// item是本地临时变量，但是在for循环执行后，仍然可以访问
for (var item : list) { x = x + item; }
```

##### while

```java
while (x lt 10) { x = x + 2; }
do { x = x + 2; } while (x lt 10)
```

##### continue/break

支持类似java的continue和break语句



### Example

```java
// Create or retrieve an engine
JexlEngine jexl = new JexlBuilder().create();

// Create an expression
String jexlExp = "foo.innerFoo.bar()";
JexlExpression e = jexl.createExpression( jexlExp );

// Create a context and add data
JexlContext jc = new MapContext();
jc.set("foo", new Foo() );

// Now evaluate the expression, getting the result
Object o = e.evaluate(jc);
```



