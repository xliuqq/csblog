# Java 代码测试

## JUnit

Java 单元测试。Junit 5，注释而不需继承通过反射机制

### 命令运行

  - Java：`$ /path/to/java –cp /path/to/junit.jar:. [classes to test]`
  - MVN：`mvn test –Dtest=*SystemTest` ，支持通配符    

### 注解

执行顺序，BeforeClass -> constructor -> Before -> Test -> After -> AfterClass

- @BeforeClass

  - static，类使用之前，只执行一次

- @Before

  - 在对象constructor调用之后，多少个Test，执行多少次

- @Test

  - Test函数之间是互相独立的，不保证执行顺序；**每个Test都会创建一个对象的实例；**

- @Timeout(value = 10)，表明如果Test函数不在10 s内完成，则测试失败；

- @After

  - 在对象析构之前，多少个Test，执行多少次

- @AfterClass

  - public static void；类析构之前，只执行一次

- @Ignore

  - 对于@Test的函数，表明该函数不进行测试
- @Runwith(Parameterized.class)，默认是Junit4
  - 使用RunWith注解改变JUnit的默认执行类，并实现自已的Listener在平时的单元测试;JUnit允许用户指定其它的单元测试执行类，只需要我们的测试执行类继承类org.junit.runners.BlockJUnit4ClassRunner（继承基础类Runner也可），如Spring的执行类SpringJUnit4ClassRunner
- @FixMethodOrder，指定Test的执行顺序；
- @Tag 测试用例分组；
- @TestFactory ：跟 DynamicTest 结合用作对生成动态测试用例，用于将数据的输入和输出和测试逻辑分开；

### 断言

org.junit.Assert，第一个参数可以是出错时打印到信息（ErrorMessage, say what shoule happen）

- assertTrue，assertEquals，assertNotNull，assertArrayEquals，fail
- assertThrows：断言应该抛出什么异常；

## [AssertJ](https://assertj.github.io/doc/)

AssertJ - Fluent Assertions for Java

```java
assertThat(frodo.getAge()).as("check %s's age", frodo.getName()).isEqualTo(33);
```

## JBehave

BDD：Behavior-Driven Development，行为驱动开发，Given->When->Then，[JBehave](http://jbehave.org/reference/stable/getting-started.html)；



## Mock

单元测试时，发现测试方法会引用外部依赖对象，通过Mock模拟外部依赖的对象；模拟对象(Mock Object)

**Mockito** 是 。

```java
Mockito.when(mock(ClassA.class).method()).thenReturn().thenReturn();
```

**Mockito**：Java 的一种Mock实现，**不能模拟静态方法、构造函数、私有函数和final类**

- `ClassA a = mock(ClassA.class) `或者 通过@Mock注解（注解需要@RunWith(MockitoJunitRunner.class)或在setUp()中调用MockitoAnnotations.initMocks()）
- `Mockito.when(mock.method()).thenReturn(value)`：设定mock对象某个方法调用时的返回值，可以多个thenReturn
- `when(mock.method()).thenThrow(new     RuntimeException())`
- `thenCallRealMethod()；`
- 对void方法预设，doNothing().when(mock.method)或者doThrow，或doNohting().doThrow()；
- 方法的参数可以使用参数模拟器，如anyInt()匹配任意int类型参数，anyString()，anySet()，使用了参数匹配器，所有的参数都需要由匹配器提供；
- mock对象只能调用设定的方法，而spy是对真正对象监控，可以使用设定的方法以及对象原生的方法；
- @injectMock会创建一个实例，并将@Mock的对象传入该实例中作成员变量；



**PowerMock**：对 Mockito 支持以及通过改字节码**支持模拟静态方法、构造函数、私有函数和final类**

- 使用静态方法等时，@RunWith(PowerMockRunner.class) 和 @PrepareForTest(被测试的类）
- 用法和Mockito一致，构造函数 whenNew()
- 测试时的字节码和平时编译出来的字节码不一致



## 代码覆盖率

