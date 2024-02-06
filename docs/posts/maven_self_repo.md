---
date: 2023-12-31
readtime: 10
categories:
  - maven
---

# Maven 自定义仓库

当自己开发一个工具包，然后另一个项目要引用时，因此需要将 jar 包放到可访问的公网上：

- Maven 官方 repo，如使用 Sonatype OSSRH，但其注册复杂，此次不进行介绍；

- 第三方公共仓库，如 JitPack, Github/Gitee等，后面进行介绍。



<!-- more -->

## JitPack 作为Maven仓库

> 只能根据项目地址自动生成 group id 和 artificat id，不会根据pom.xml中定义的生成。

### 构建 jar 包

根据 Tag / Release 来：

- 输入项目地址，点击'Look up' 会自动构建；
- `Log` 查看构建历史

![image-20231231182808165](pics/maven_jitpack.png)



### 使用Jar包

pom.xml 中配置

```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>

<dependency>
    <groupId>com.gitee.cs-open-project</groupId>
    <artifactId>annotations-for-readme</artifactId>
    <version>v1.0</version>
</dependency>
```



## Github 项目作为Maven仓库

> Gitee 也可以作为 Maven 仓库，但是第二步"上传 Deploy 输出"，无法通过maven插件执行，需要手动上传。



### 指定构建Deploy的输出位置

```xml
 <plugin>
    <!--用于将构建产物（通常是JAR文件、WAR文件等）部署到输出位置-->
    <artifactId>maven-deploy-plugin</artifactId>
    <version>2.8.1</version>
    <configuration>
		<altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo</altDeploymentRepository>
    </configuration>
</plugin>
```



### 上传 Deploy 输出

m2的`settings.xml`配置 server

```xml
<servers>
  <server>
    <id>xliuqq-github</id><!-- 自定义的id。记住此id, 后续发布jar包时要使用 -->
    <!-- token 需要具备 repo 的读写，和 user email 权限-->
    <password>your-token</password>
  </server>
</servers>
```

待发布的工程的 pom.xml 中配置，将包发布到 github 的仓库中

```xml
<!--用于发布到Github 指定仓库-->
<plugin>
    <groupId>com.github.github</groupId>
    <artifactId>site-maven-plugin</artifactId>
    <version>0.12</version>
    <executions>
        <execution>
            <goals>
                <goal>site</goal>
            </goals>
            <phase>deploy</phase>
            <configuration>
                <!--服务名（settings.xml中配置内容，上述配置的server的id）-->
                <server>xliuqq-github</server>
                <!-- 提交消息（commit message）-->
                <message>Maven artifacts for [${project.artifactId}-${project.version}]</message>
                <!-- 上传网站的位置（远程仓库名） -->
                <repositoryName>maven-repository</repositoryName>
                <!--组织或用户名-->
                <repositoryOwner>xliuqq</repositoryOwner>
                <!-- 使用合并或覆盖内容 -->
                <merge>true</merge>
                <!--分支（固定写法：refs/heads/ + 你的分组）-->
                <branch>refs/heads/main</branch>
                <!--源文件地址，跟上一步的deploy输出位置对应-->
                <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory>
                <!-- 包含outputDirectory标签内填的文件夹中的所有内容 -->
                <includes>
                    <include>**/*</include>
                </includes>
            </configuration>
        </execution>
    </executions>
</plugin>
```



### 使用 jar

pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.xliu.cs.projects</groupId>
        <artifactId>annotations-for-doc</artifactId>
        <version>1.0</version>
    </dependency>
</dependencies>

<repositories>
  <repository>
    <id>maven-github</id>    <!--此id需要与setting.xml中的配置，但不用跟第一步中上传的server名一致-->
    <!-- github 访问路径 -->
    <url>http://raw.github.com/xliuqq/maven-repository/main</url>
    <!-- 如果 github 无法访问（或者配置代理仍出现`Unknown host 请求的名称有效，但是找不到请求的类型的数据`问题，可通过gitee同步gihub项目，然后使用 gitee 地址 -->
    <!-- <url>https://gitee.com/luckyQQQ/maven-repository/raw/main</url> -->
  </repository>
</repositories>
```

如果配置了 mirror *，则需要忽略这个 repository-id。

- 

```xml
<!-- 配置1：
  注意: <mirrorOf>*,!your-github</mirrorOf>
  原本: <mirrorOf>*</mirrorOf> 
-->
<mirror><!--此处使用阿里云镜像，增加!your-repository-id，跟上面保持一致-->
  <id>nexus-aliyun</id>
  <mirrorOf>*,!maven-github</mirrorOf>
  <name>Nexus aliyun</name>
  <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
```

