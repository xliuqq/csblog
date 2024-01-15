---
date: 2024-01-15
readtime: 20
categories:
  - maven
---



# Maven 插件开发

在开发[通过包/类/方法注解自动生成README](https://gitee.com/cs-open-project/annotations-for-readme)的功能时，在使用时，如果使用 Jar 依赖的方式，需要为每个工程定义一个 Main 方法，并且需要手动调用。

因此想通过开发Maven 插件，在compile阶段自动调用并生成 README，因此便研究如何开发 Maven 插件。



<!-- more -->

## 开发

> 官方参考文档：https://maven.apache.org/plugin-developers/index.html

pom.xml 配置，最新版的配置见：

- maven核心版本：https://mvnrepository.com/artifact/org.apache.maven/maven-plugin-api
- maven插件版本：https://mvnrepository.com/artifact/org.apache.maven.plugin-tools/maven-plugin-tools-api

```xml
<dependencies>   
    <dependency>
        <groupId>org.apache.maven.plugin-tools</groupId>
        <artifactId>maven-plugin-annotations</artifactId>
        <version>3.9.0</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <!-- 获取 MavenProject 和 MavenSession 时需要引用的包，maven core 和 plugin 大版本一致，但小版本不完全一致 -->
        <groupId>org.apache.maven</groupId>
        <artifactId>maven-core</artifactId>
        <version>3.9.5</version>
        <scope>provided</scope>
    </dependency>
	<dependency>
        <groupId>org.apache.maven</groupId>
        <artifactId>maven-plugin-api</artifactId>
        <version>3.9.5</version>
        <scope>provided</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-plugin-plugin</artifactId>
            <version>3.9.0</version>
        </plugin>
    </plugins>
</build>
```



### 生命周期

开发插件，需要继承 `AbstractMojo` 并实现 execute 方法；

- `@Mojo`的`name` 是 plugin xml中 execution 中的 goal 的可选项

```java
@Mojo(name = "generate-doc", defaultPhase = LifecyclePhase.COMPILE)
public class AnnotationMojo extends AbstractMojo {
	public void execute() throws MojoExecutionException {
        getLog().info("Hello, world.");
    }
}
```



### 参数

`@Parameter` 注解定义参数：

- `defaultValue` 为 `""` 时，其在运行时的值为 `null`；
- 参数直接表达式，如`${project.basedir}`表示项目的根路径

```java
@Parameter(name = "scanPkgPrefix", required = true， threadSafe=true)
private String scanPkgPrefix;

@Parameter(name = "outputFilePath", defaultValue = "./README.md")
private String outputFilePath;

@Parameter(name = "skipPkgs", defaultValue = "")
private String skipPkgs;
```

特殊的参数：

```java
// 当前插件处理的项目，多maven项目时，插件会被调用多次
@Parameter(defaultValue = "${project}", required = true, readonly = true)
MavenProject currentProject;

// maven session，可获取所有的maven项目以及项目间的依赖关系
@Parameter(defaultValue = "${session}", readonly = true, required = true)
private MavenSession session;
```



### 类加载器

> [Maven – Guide to Maven Classloading (apache.org)](https://maven.apache.org/guides/mini/guide-maven-classloading.html)

通过在插件的`execute`函数中，打印`java.class.path`属性，类加载器是`org.codehaus.plexus.classworlds.realm.ClassRealm`

- 父类加载是 `AppClassLoader`

```shell
java.class.path -> C:\Program Files\JetBrains\IntelliJ IDEA 2023.3.2\plugins\maven\lib\maven3\boot\plexus-classworlds-2.7.0.jar
```



#### maven core依赖的jar

开发插件时，使用 maven core 的依赖插件时，如 guava，common-lang3，在实际使用插件时会出现类不存在的情况：

- maven 软件的 `lib`目录下，存在这些第三方依赖 jar 包；
- 但在插件运行时，其类加载器没有这些 jar 包；？？



#### 项目源码获取

maven 插件开发时，其类加载不包含工程代码路径，因此无法通过反射获取类的相关信息。

- 获取项目代码的类的相关信息，可以通过自定义类加载器实现，下面是示例：

```java
public class AnnotationMojo extends AbstractMojo {
    @Parameter(defaultValue = "${project}", required = true, readonly = true)
    MavenProject project;

    public void execute() throws MojoExecutionException {
        // 将编译后的class路径加到 classpath 中
        List<String> classpathElements = project.getCompileClasspathElements();
        urls = new URL[classpathElements.size()];
        for (int i = 0; i < classpathElements.size(); ++i) {
            urls[i] = new File(classpathElements.get(i)).toURI().toURL();
        }
        // 通过自定义的类加载器进行类的加载
        ClassLoader classloader = new URLClassLoader(urls, getClass().getClassLoader());
        Class.forName("org.xliu.cs.projects.ClassA", true, classloader);
    }
}
```



### 日志

> https://maven.apache.org/ref/current/maven-embedder/logging.html

maven 3.5 之后，可以直接使用 Slf4j 的 API 进行日志打印，maven 使用时在控制台会进行打印。

### 部署

maven install 部署到本地；

maven deploy 部署到服务器；

## 调试

> 当插件名称符合 XXX-maven-plugin 时，mvn 使用时，只需用 XXX 即可，后面可以省略。

`mvnDebug` 命令调试，程序会自动分配8000 Listen端口，会等待 Remote 连接后，才会继续执行：

- `mvnDebug` 可以用 IDEA 内置的 maven，一般会在`plugins\maven\lib\maven3\bin`目录下；

- 命令示例：`mvnDebug annotations-for-doc:generate-doc@{id}`
  - 必须指定 id，不然命令行执行时，不会读取 configuration 的配置

```xml
<plugin>
    <groupId>org.xliu.cs.projects</groupId>
    <artifactId>annotations-for-doc-maven-plugin</artifactId>
    <version>1.0.1</version>
    <executions>
        <execution>
            <!-- 可以配置多个 execution，只要 id 不一样即可，id只是标识符（没有其他含义） -->
            <id>generate</id>
            <goals>
                <!-- 匹配@Mojo 注解的name -->
                <goal>generate-doc</goal>
            </goals>
            <configuration>
                <scanPkgPrefix>org.xliu.cs.jvm</scanPkgPrefix>
                <outputFilePath>${project.basedir}/README.md</outputFilePath>
            </configuration>
        </execution>
    </executions>
</plugin>
```

然后 IDEA 中打开插件源码，用 Remote 模式进行断点调试（本地mvn仓库需要包含插件的源码，否则无法 debug）；