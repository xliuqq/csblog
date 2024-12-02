# 注解

> 注解定义时，不可以继承其它的注解或接口；

## 元注解

四种元注解，java.lang.annotation，专门注释其他的注解：

- @Documented：注解是否将包含在JavaDoc中

- @Retention：什么时候使用该注解，注解的生命周期，RetentionPolicy

  - SOURCE：编译阶段丢弃，注解在编译结束无意义，如@Override,       @suppressWarnings；
  - CLASS：类加载时候丢弃，在字节码文件的处理中有用，默认方式；
  - RUNTIME：不丢弃，运行期也保留注解，可以使用反射机制读取注解信息；自定义注解使用；

- @Target：注解用于什么地方，ElementType.{TYPE, METHOD,…}
- @Inherited：是否允许子类继承该注解，只对`类`生效；
  - **如果元注解Inherited修饰的其他注解，修饰了除类之外的其他程序元素，那么继承性将会失效**。
  - 方法和属性上注解的继承，忠实于方法/属性继承本身，客观反映方法/属性上的注解。


@interface用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是基本类型、Class、String、enum）。可以通过default来声明参数的默认值。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Action {
    String input() default "";
    FaultAction[] fault() default { };
}
```



### 常用的注解

#### @GuardedBy

@GuardedBy("mResourcesManager")

是一个同步锁，只有在**线程持有mResourcesManager对象锁**时，才能使用这写变量。



## 注解组合

> Spring 的能力，并非是 Java 自身的能力，更详细的内容见 Spring 的注解组合。

```java
// 通过该函数能够获取到被注解的注解
AnnotatedElementUtils.getMergedAnnotation(Element.class, Test2.class);

@Target({ ANNOTATION_TYPE, FIELD, TYPE })
@Retention(RUNTIME)
@interface Test2 {
    String test2() default "test2";
}

@Target({ ANNOTATION_TYPE, FIELD, TYPE })
@Retention(RUNTIME)
@Test2
@interface Test3 {
    String tset3() default "test3";
}
 
@Test3
static class Element {}
```



## Processor

`javax.annotation.processing.Processor`

注解处理器(Annotation Processor)是 **javac 内置的一个用于编译时**扫描和处理注解(Annotation)的工具，常用的用法就是在编译期间获取相关注解数据，然后动态生成`.java`源文件，比如lombok。

在编译的时候，javac会去找所有jar包及项目（模块）里`resource/META-INF/services/javax.annotation.processing.Processor`这个文件中配置的类信息，记住是**类信息**，它会通过classloader去加载这个类，此时项目（模块）中的文件因为是在编译期，尚未生成class文件，也就找不到对应的类，解决方法：

- 通过`maven-resource`插件，编译完之后再进行 copy-resource；

简单案例见代码库

- [GetterSetterProcessor](https://gitee.com/luckyQQQ/lifelearning/tree/master/java/jvm/annotation/core)
- [demo](https://gitee.com/luckyQQQ/lifelearning/tree/master/java/jvm/annotation/demo)

### 编译期间修改语法树

示例：实现 GettetSetter 注解（依赖 tools.jar）；

- 直接修改语法树，也可以先生成源文件，再进行编译；

```java
// 根据字符串获取Name
private Name getNameFromString(String s) { return names.fromString(s); }

// 创建 域/方法 的多级访问, 方法的标识只能是最后一个 例如： java.lang.System.out.println
private JCTree.JCExpression memberAccess(String components) {
    String[] componentArray = components.split("\\.");
    JCTree.JCExpression expr = treeMaker.Ident(getNameFromString(componentArray[0]));
    for (int i = 1; i < componentArray.length; i++) {
        expr = treeMaker.Select(expr, getNameFromString(componentArray[i]));
    }
    return expr;
}

// 创建变量语句
private JCTree.JCVariableDecl makeVarDef(JCTree.JCModifiers modifiers, String name, JCTree.JCExpression vartype, JCTree.JCExpression init) {
    return treeMaker.VarDef(
        modifiers,
        getNameFromString(name), //名字
        vartype, //类型
        init //初始化语句
    );
}
// 给变量赋值
private JCTree.JCExpressionStatement makeAssignment(JCTree.JCExpression lhs, JCTree.JCExpression rhs) {
return treeMaker.Exec(
	treeMaker.Assign(lhs, rhs)
	);
}

// 声明变量并赋值（利用以上包装的方法）  生成语句为：String xiao = "methodName";
JCTree.JCVariableDecl var = makeVarDef(treeMaker.Modifiers(0), "xiao", memberAccess("java.lang.String"), treeMaker.Literal("methodName"));

// 生成语句为：xiao = "-Binary operator one" + "-Binary operator two";
treeMaker.Exec(
	treeMaker.Assign(treeMaker.Ident(getNameFromString("xiao")),
		treeMaker.Binary(
			JCTree.Tag.PLUS,
            treeMaker.Literal("-Binary operator one"),
            treeMaker.Literal("-Binary operator two")
		))
);

// 声明整型变量并赋值 生成语句为：Integer zhen = 1;
makeVarDef(treeMaker.Modifiers(0), "zhen", memberAccess("java.lang.Integer"), treeMaker.Literal(1));

// 生成语句：System.out.println(xiao)
JCTree.JCExpressionStatement printVar = treeMaker.Exec(treeMaker.Apply(
    List.of(memberAccess("java.lang.String")),//参数类型
    memberAccess("java.lang.System.out.println"),
    List.of(treeMaker.Ident(getNameFromString("xiao")))
	)
);
```

