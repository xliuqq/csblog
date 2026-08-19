# Maven

## 配置

### 依赖管理

冲突解决

在配置文件pom.xml中先声明要使用哪个版本的相应jar包，声明后其他版本的jar包一律不依赖。（type和import表示从该pom文件中继承依赖管理）

```xml 
<properties>
    <spring.version>4.2.4.RELEASE</spring.version>
    <hibernate.version>5.0.7.Final</hibernate.version>
    <struts.version>2.3.24</struts.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>com.cbm.stu</groupId>
            <artifactId>maven-parent-a</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 构建

#### Maven单独构建多模块项目中的单个模块

说明：

1、可能存在的场景，多模块项目没有互相引用，那么此时可以单独构建单个项目，指定到子模块的pom.xml文件即可完成编译。

2、如果多模块项目各自都引用了，那么单独编译子模块的pom.xml文件会直接报错，解决方法就是编译父项目pom.xml。

3、如果编译父项目，那么可能会造成编译时间很慢，其中有些项目也不需要编译，解决方法如下：

Maven选项：

```shell
-pl, --projects
        Build specified reactor projects instead of all projects
-am, --also-make
        If project list is specified, also build projects required by the list
-amd, --also-make-dependents
        If project list is specified, also build projects that depend on projects on the list
```

首先切换到工程的根目录

单独构建模块jsoft-web，同时会构建jsoft-web模块依赖的其他模块

```shell
mvn install -pl jsoft-web -am
```

单独构建模块jsoft-common，同时构建依赖模块jsoft-common的其他模块 

```shell
mvn install -pl jsoft-common -am -amd
```

### Resources

- 默认会把src/main/resources下的文件和class文件一起打包到jar内部，可以通过

```xml
<build>
    <resources>  
        <resource>  
            <directory>src/main/resources</directory>  
            <includes>  
                <include>*.txt</include>  
            </includes>  
            <excludes>  
                <exclude>*.xml</exclude>  
            </excludes>  
        </resource>  
        <!-- 把src/main/resources目录下所有的文件拷贝到conf目录中 -->
        <resource>
            <directory>src/main/resources</directory>
            <targetPath>${project.build.directory}/conf</targetPath>
        </resource>
        <!-- 把lib目录下所有的文件拷贝到lib目录中
            （可能有些jar包没有办法在maven中找到，需要放在lib目录中） -->
        <resource>
            <directory>lib</directory>
            <targetPath>${project.build.directory}/lib</targetPath>
        </resource>
        <!-- 把放在根目录下的脚本文件.sh,.bat拷贝到bin目录中 -->
        <resource>
            <directory>.</directory>
            <includes>
                <include>**/*.sh</include>
                <include>**/*.bat</include>
            </includes>
            <targetPath>${project.build.directory}/bin</targetPath>
        </resource>
    </resources>  
</build>
```



## 生命周期

[ validate, **initialize**, generate-sources, process-sources, generate-resources, process-resources, **compile**, process-classes, generate-test-sources, process-test-sources, generate-test-resources, process-test-resources, test-compile, process-test-classes, **test**, prepare-package, **package**, pre-integration-test, integration-test, post-integration-test, **verify**, install, deploy ]

### 插件执行顺序

**maven对于绑定到同一phase上的多个插件的执行顺序是按照它们在pom.xml声明的顺序来的**。

如果在父 pom 中定义，或者在 profile 的 plugins 定义，需要在子pom中重新声明，以指定执行顺序。

**Parent pom.xml**

```xml
<plugins>
    <plugin>
        <groupId>groupid.maven.1</groupId>
        <artifactId>maven-plugin-1</artifactId>
        <version>1.0</version>
        <executions>
            <execution>
                <phase>package</phase>
            </execution>
        </executions>
    </plugin>
</plugins>
```

**Child pom.xml**

```xml
<plugins>
    <plugin>
        <groupId>groupid.maven.2</groupId>
        <artifactId>maven-plugin-2</artifactId>
        <version>1.0</version>
        <executions>
            <execution>
                <phase>package</phase>
            </execution>
        </executions>
    </plugin>
    <plugin>
        <groupId>groupid.maven.1</groupId>
        <artifactId>maven-plugin-1</artifactId>
    </plugin>
</plugins>
```



## 插件

### 官方清单

http://maven.apache.org/plugins/index.html



### flatten-maven-plugin（多模块版本号）

> [Releases · mojohaus/flatten-maven-plugin (github.com)](https://github.com/mojohaus/flatten-maven-plugin/releases)
>
> 用于大型的多maven模块的项目，用于版本号的管理。
>
> - 打包的时候（package、install、deploy）将生成 `.flattened-pom.xml`做为当前项目的 pom 文件

父 pom

```xml
<!-- 在 父 pom 中定义 ${revision} 变量-->
<version>${revision}</version>
<packaging>pom</packaging>

<properties>
    <revision>1.0.2</revision>
</properties>



<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>flatten-maven-plugin</artifactId>
    <version>1.5.0</version>
    <configuration>
        <!-- 对于 packaging 为 pom 类型的项目也使用生成的 .flattened-pom.xml -->
        <updatePomFile>true</updatePomFile>
        <flattenMode>resolveCiFriendliesOnly</flattenMode>
    </configuration>
    <executions>
        <execution>
            <id>flatten</id>
            <phase>process-resources</phase>
            <goals>
                <goal>flatten</goal>
            </goals>
        </execution>
        <execution>
            <id>flatten.clean</id>
            <phase>clean</phase>
            <goals>
                <goal>clean</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

子 pom

```xml
<parent>
  <version>${revision}</version>
</parent>
<!-- 子模块的版本应该使用父工程的版本，单独设置版本的话会导致版本混乱，即不设置 version -->

<dependency>
    <groupId>org.apache.maven.ci</groupId>
    <artifactId>child2</artifactId>
	<!-- 以来其它子模块时，使用 ${project.version} 或者 ${revision} 都可以 -->
    <version>${project.version}</version>
</dependency>
```



### dependency-plugin

将依赖的包递归下载，并拷贝到指定目录

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}</outputDirectory>
                <overWriteReleases>true</overWriteReleases>
                <overWriteSnapshots>true</overWriteSnapshots>
                <overWriteIfNewer>true</overWriteIfNewer>
                <useSubDirectoryPerType>true</useSubDirectoryPerType>
                <!--                        <includeArtifactIds>-->
                <!--                            Rserve,breeze_${scala.binary.version},breeze-natives_${scala.binary.version}-->
                <!--                        </includeArtifactIds>-->
                <silent>true</silent>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### maven-jar-plugin

打包的插件。

```xml
<!-- 用于生成jar包的plugin -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.6</version>
    <configuration>
        <!-- 把生成的jar包放在lib目录下（和其他所有jar包一起） -->
        <outputDirectory>${project.build.directory}/lib</outputDirectory>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
            </manifest>
        </archive>
        <excludes>
            <!-- 排除掉一些文件,不要放到jar包中，
               这里是为了排除掉src/main/resources中的文件（它们应该放到conf目录）
               这里只能指定要排除的目标文件，而不能指定源文件，虽然不够完美，但是基本能达到目的。 -->
            <exclude>*.xml</exclude>
            <exclude>*.properties</exclude>
        </excludes>
    </configuration>
</plugin>
```

### 

### assembly plugin

http://maven.apache.org/plugins/maven-assembly-plugin/usage.html

- maven-assembly-plugin，支持自定义的打包结构，也可以定制依赖项等。

  Maven预先定义好的描述符有bin，src，project，jar-with-dependencies等。比较常用的是jar-with-dependencies，它是将所有外部依赖JAR都加入生成的JAR包

```xml
    <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
            <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>
        </configuration>
        <executions>
            <execution>
                <id>make-assembly</id>
                <phase>package</phase>
                <goals>
                    <goal>single</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
```

### shade plugin

- maven-jar-plugin，默认的打包插件，用来打普通的project JAR包；
- maven-shade-plugin，用来打可执行JAR包，也就是所谓的fat JAR包；
- maven-assembly-plugin，支持定制化打包方式，例如 apache 项目的打包方式


**使用 maven-shade-plugin 时需要小心处理字符串**

下面的示例将jedis 打进了 jar 包中，并且将包名前缀改为 `scyuan.maven.shaded.redis`

```xml
<plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-shade-plugin</artifactId>
     <version>3.1.1</version>
     <configuration>
         <artifactSet>
             <includes>
                 <include>redis.clients:jedis</include>
                 <include>org.apache.commons:commons-pool2</include>
             </includes>
         </artifactSet>
         <filters>
             <filter>
                 <artifact>*:*</artifact>
                 <excludes>
                     <exclude>META-INF/*.SF</exclude>
                     <exclude>META-INF/*.DSA</exclude>
                     <exclude>META-INF/*.RSA</exclude>
                 </excludes>
             </filter>
         </filters>
         <relocations>
             <relocation>
                 <pattern>redis</pattern>
                 <shadedPattern>scyuan.maven.shaded.redis</shadedPattern>
             </relocation>
             <relocation>
                 <pattern>org.apache.commons</pattern>
                 <shadedPattern>scyuan.maven.shaded.org.apache.commons</shadedPattern>
             </relocation>
         </relocations>
     </configuration>
     <executions>
         <execution>
             <phase>package</phase>
             <goals>
                 <goal>shade</goal>
             </goals>
         </execution>
     </executions>
</plugin>

```

### maven-checkstyle-plugin

java代码风格的checkstyle

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.1.1</version>
    <dependencies>
        <dependency>
            <groupId>com.puppycrawl.tools</groupId>
            <artifactId>checkstyle</artifactId>
            <version>8.36</version>
        </dependency>
    </dependencies>
    <configuration>
        <configLocation>Build/hongcheng_checkstyle.xml</configLocation>
    </configuration>
</plugin>
```

### maven-clean-plugin

mvn clean时调用的就是这个插件，主要作用就是清理构建目录下得全部内容,构建目录默认是target；

构建时需要清理构建目录以外的文件，比如制定的日志文件，这时候就需要配置`<filesets>`来实现了。

### maven-compile-plugin

指定编译用的java版本号

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
    </configuration>
</plugin>
```

### maven-dependency-plugin

dependency:copy：将一系列在此插件内列出的artifacts ，将他们copy到一个特殊的地方，重命名或者去除其版本信息。

这个可以解决远程仓库存在但是本地仓库不存在的依赖问题，copy操作可以用来将某个（些）maven artifact(s)拷贝到某个目录。

### maven-enforcer-plugin

jar冲突，进行mvn clean package的时候，会在console中打印出来冲突的jar版本和其父pom。

### maven-install-plugin

> The Install Plugin is used during the install phase to add artifact(s) to the local repository.
>
> The local repository is the local cache where all artifacts needed for the build are stored. By default, it is located within the user's home directory (~/.m2/repository) but the location can be configured in ~/.m2/settings.xml using the <localRepository> element.

### maven-deploy-plugin

打包发布到maven仓库中，可以配置发布到私服中；

### maven-surefire-plugin(java test)

Maven所做的只是在构建执行到特定生命周期阶段的时候，通过插件来执行JUnit或者TestNG的测试用例；

test阶段与 maven-surefire-plugin 的test目标相绑定了， 这是一个内置的绑定。

```xml
 <plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-surefire-plugin</artifactId>
     <version>3.0.0-M5</version>
     <configuration>
         <argLine>
             -javaagent:${settings.localRepository}/com/alibaba/testable/testable-agent/${testable.version}/testable-agent-${testable.version}.jar
         </argLine>
     </configuration>
</plugin>
```

### mvn-scalafmt

在validate阶段对scala代码进行格式化。

### build-helper-maven-plugin

Maven默认只允许指定一个主Java代码目录和一个测试Java代码目录。 

虽然这其实是个应当尽量遵守的约定，但偶尔你还是会希望能够指定多个源码目录（例如为了应对遗留项目），build-helper-maven-plugin的add-source目标就是服务于这个目的，通常它被绑定到默认生命周期的generate-sources阶段以添加额外的源码目录。

### exec-maven-plugin

指定在maven的某个阶段(phase)执行一些命令，如运行java程序，R install等；

### jacoco-maven-plugin（Java代码覆盖率）

Testable Mock插件

```xml
<dependency>
    <groupId>com.alibaba.testable</groupId>
    <artifactId>testable-all</artifactId>
    <version>0.6.0</version>
    <scope>test</scope>
</dependency>

<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration>
            <argLine> @{argLine} -javaagent:${settings.localRepository}/com/alibaba/testable/testable-agent/${testable.version}/testable-agent-${testable.version}.jar</argLine>
        </configuration>
    </plugin>
</plugins>
```



https://mvnrepository.com/artifact/org.jacoco/jacoco-maven-plugin

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.7</version>
</plugin>
```

### scalastyle-maven-plugin

Scala style 检查，

### scala plugin（scala-maven, scala-test-maven）

```xml
<plugin>
    <groupId>net.alchim31.maven</groupId>
    <artifactId>scala-maven-plugin</artifactId>
                <version>4.8.1</version>
    <executions>
        <execution>
            <id>eclipse-add-source</id>
            <goals>
                <goal>add-source</goal>
            </goals>
        </execution>
        <!-- Run scala compiler in the process-resources phase, so that dependencies on scala classes can be resolved later in the (Java) compile phase -->
        <execution>
            <id>scala-compile-first</id>
            <phase>process-resources</phase>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>

        <!-- Run scala compiler in the process-test-resources phase, so that dependencies on scala classes can be resolved later in the (Java) test-compile phase -->
        <execution>
            <id>scala-test-compile</id>
            <phase>process-test-resources</phase>
            <goals>
                <goal>testCompile</goal>
            </goals>
        </execution>
        <execution>
            <id>attach-scaladocs</id>
            <phase>verify</phase> <!-- 默认是package阶段!-->
            <goals>
                <goal>doc-jar</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <scalaVersion>${scala.version}</scalaVersion>
        <checkMultipleScalaVersions>true</checkMultipleScalaVersions>
        <failOnMultipleScalaVersions>true</failOnMultipleScalaVersions>
        <!--  compile both Scala and Java sources !-->
        <recompileMode>incremental</recompileMode>
        <args>
            <arg>-unchecked</arg>
            <arg>-deprecation</arg>
            <arg>-feature</arg>
            <arg>-explaintypes</arg>
            <arg>-Yno-adapted-args</arg> <!-- !-->
            <arg>-target:jvm-1.8</arg>
        </args>
        <jvmArgs>
            <jvmArg>-Xms1024m</jvmArg>
            <jvmArg>-Xmx1024m</jvmArg>
            <jvmArg>-XX:ReservedCodeCacheSize=${CodeCacheSize}</jvmArg>
        </jvmArgs>
        <javacArgs>
            <javacArg>-source</javacArg>
            <javacArg>${java.version}</javacArg>
            <javacArg>-target</javacArg>
            <javacArg>${java.version}</javacArg>
            <javacArg>-Xlint:all,-serial,-path,-try</javacArg>
        </javacArgs>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.0</version>
    <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
    </configuration>
</plugin>

<!-- Scalatest runs all Scala tests -->
<plugin>
    <groupId>org.scalatest</groupId>
    <artifactId>scalatest-maven-plugin</artifactId>
    <version>2.0.0</version>
    <configuration>
        <reportsDirectory>${project.build.directory}/surefire-reports
        </reportsDirectory>
        <junitxml>.</junitxml>
        <filereports>SparkTestSuite.txt</filereports>
        <argLine>-ea -Xmx4g -Xss4m -XX:ReservedCodeCacheSize=${CodeCacheSize}
        </argLine>
        <stderr/>
        <environmentVariables>
            <!--
       Setting SPARK_DIST_CLASSPATH is a simple way to make sure any child processes
       launched by the tests have access to the correct test-time classpath.
     -->
            <SPARK_DIST_CLASSPATH>${test_classpath}</SPARK_DIST_CLASSPATH>
            <SPARK_PREPEND_CLASSES>1</SPARK_PREPEND_CLASSES>
            <SPARK_SCALA_VERSION>${scala.binary.version}</SPARK_SCALA_VERSION>
            <SPARK_TESTING>1</SPARK_TESTING>
            <JAVA_HOME>${test.java.home}</JAVA_HOME>
        </environmentVariables>
        <systemProperties>
            <log4j.configuration>file:src/test/resources/log4j.properties
            </log4j.configuration>
            <spark.test.home>${spark.test.home}</spark.test.home>
            <spark.testing>1</spark.testing>
        </systemProperties>
    </configuration>
    <executions>
        <execution>
            <id>test</id>
            <goals>
                <goal>test</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

