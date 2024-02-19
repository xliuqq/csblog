---
date: 2024-01-15
readtime: 20
categories:
  - spark
---



# Spark 使用 Yarn Rest 提交

> https://community.cloudera.com/t5/Community-Articles/Starting-Spark-jobs-directly-via-YARN-REST-API/ta-p/245998 这篇文章是Spark 1.6， 已经过时，只能作为基本的参考，具体还是要阅读Spark on Yarn的提交代码。
>
> - 本文基于 Spark 2.4.8 和 Hadoop 3.2 进行验证。



将 Spark 作业提交到 Yarn上时，只能通过命令行 spark-submit 进行操作，本文通过解析 spark-submit 的源码，探究如何使用 Yarn Rest API 进行提交 Spark 作业（仅 cluster 模式，因 client 模式 driver 运行在 client 中而不是 AM 中）。



一句话总结：**还是用命令行调用`spark-submit` ！**



<!-- more -->



## 1. Yarn Rest API

### 1.1 创建Yarn 应用

```
POST http://rm-http-address:port/ws/v1/cluster/apps/new-application
```

返回体示例：`application-id` 和单个应用申请的不同资源的最大数量

```json
{
    "application-id": "application_1632703810135_0002",
    "maximum-resource-capability": {
        "memory": 20480,
        "vCores": 4,
        "resourceInformations": {
            "resourceInformation": [
                {
                    "maximumAllocation": 9223372036854775807,
                    "minimumAllocation": 0,
                    "name": "memory-mb",
                    "resourceType": "COUNTABLE",
                    "units": "Mi",
                    "value": 20480
                },
                {
                    "maximumAllocation": 9223372036854775807,
                    "minimumAllocation": 0,
                    "name": "vcores",
                    "resourceType": "COUNTABLE",
                    "units": "",
                    "value": 4
                },
                {
                    "maximumAllocation": 9223372036854775807,
                    "minimumAllocation": 0,
                    "name": "yarn.io/gpu",
                    "resourceType": "COUNTABLE",
                    "units": "",
                    "value": 3
                }
            ]
        }
    }
}
```

### 1.2. 提交Yarn作业

```
POST http://rm-http-address:port/ws/v1/cluster/apps
```

请求体示例：具体字段见（https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/ResourceManagerRest.html#Cluster_Applications_API.28Submit_Application.29）

```json
{
  "application-id": "application_1632703810135_0002",
  "application-name": "Spark_Yarn_Rest",
  "queue": "default",
  "priority": 10,
  "am-container-spec": {
    // Location of the resource to be localized
    "local-resources": {
       "entry": [
         {
           "key":"AppMaster.jar",
           "value": {
             "resource": "hdfs://hdfs-namenode:9000/user/testuser/DistributedShell/demo-app/AppMaster.jar",
             // options: "ARCHIVE", "FILE", and "PATTERN"
             "type" : "FILE",
             // options are "PUBLIC", "PRIVATE", and "APPLICATION"
             "visibility": "APPLICATION",
             // Size of the resource to be localized
             "size": 43004,
             // Timestamp of the resource to be localized
             "timestamp": 1405452071209
           }
         }
       ]
    },
    "environment": {
       "entry": [{
            "key": "DISTRIBUTEDSHELLSCRIPTTIMESTAMP",
            "value": "1405459400754"
          }, {
            "key": "CLASSPATH",
            "value": "{{CLASSPATH}}<CPS>./*<CPS>{{HADOOP_CONF_DIR}}<CPS>{{HADOOP_COMMON_HOME}}/share/hadoop/common/*<CPS>{{HADOOP_COMMON_HOME}}/share/hadoop/common/lib/*<CPS>{{HADOOP_HDFS_HOME}}/share/hadoop/hdfs/*<CPS>{{HADOOP_HDFS_HOME}}/share/hadoop/hdfs/lib/*<CPS>{{HADOOP_YARN_HOME}}/share/hadoop/yarn/*<CPS>{{HADOOP_YARN_HOME}}/share/hadoop/yarn/lib/*<CPS>./log4j.properties"
          }, {
            "key": "DISTRIBUTEDSHELLSCRIPTLEN",
            "value": "6"
          }, {
            "key": "DISTRIBUTEDSHELLSCRIPTLOCATION",
            "value": "hdfs://hdfs-namenode:9000/user/testuser/demo-app/shellCommands"
          }
        ]
    },
    "commands": {
      "command": "{{JAVA_HOME}}/bin/java -Xmx10m org.apache.hadoop.yarn.applications.distributedshell.ApplicationMaster --container_memory 10 --container_vcores 1 --num_containers 1 --priority 0 1><LOG_DIR>/AppMaster.stdout 2><LOG_DIR>/AppMaster.stderr"
    },
    "service-data": null,
    "credentials": null,
    "application-acls": null
  },
  "unmanaged-AM": false,
  "max-app-attempts": 3,
  "resource": {
     "memory":1024,
     "vCores":1,
     "yarn.io/gpu": 1
  },
  "application-type": "Spark",
  "keep-containers-across-application-attempts": true,
  "application-tags": {
    
  },
  "log-aggregation-context": {
    
  },
  "attempt-failures-validity-interval": 3600000,
  "reservation-id": null,
  "am-black-listing-requests": null  
}
```



## 2. Spark 提交原理

当前只讨论`Yarn Cluster`模式，**Client模式下Driver在Client中，无法直接提供Yarn Rest服务**；

- 客户端：`org.apache.spark.deploy.yarn.Client`

- AppMaster: `org.apache.spark.deploy.yarn.ApplicationMaster`

主要目标：

- 构造AppMaster的命令行和相关配置参数（以scala提交jar包形式，其它模式可能含有不同的操作）；
- 其它的python提交，kerberos配置等，都是额外的local resource配置，原理大致一样；



### 2.1 spark-submit.sh 入口 

- 设置ENV `SPARK_HOME`；
- 设置ENV `PYTHONHASHSEED`为`0`；

- 执行如下类似的命令

```shell
exec /home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/bin/spark-class org.apache.spark.deploy.SparkSubmit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --supervise \
--executor-memory 2G --total-executor-cores 3 \
examples/jars/spark-examples_2.11-2.4.8.jar 1000
```

- 在`spark-class`里，执行的操作
  - 设置ENV `SPARK_CONF_DIR`；
  - 将`spark-env.sh`中设置的变量，全部export；
  - 设置ENV `SPARK_SCALA_VERSION`；
  - 执行如下命令，获取启动命令


```shell
/opt/tmp_workspace/jdk1.8.0_201/bin/java -Xmx128m -cp \
'/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/jars/*' \
org.apache.spark.launcher.Main org.apache.spark.deploy.SparkSubmit \
--class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster \
--supervise --executor-memory 2G --total-executor-cores 3 \
examples/jars/spark-examples_2.11-2.4.8.jar 1000
```

- `Main`中，针对`SparkSubmit`，执行的操作：
  - 构建CLASSPATH，添加`spark.driver.extraClasspath`、`$SPARK_CONF_DIR`、`$SPARK_HOME/jars`, `$HADOOP_CONF_DIR`、`$YARN_CONF_DIR`；
  - 构建之后的执行命令如下所示：


```shell
exec /opt/tmp_workspace/jdk1.8.0_201/bin/java -cp '/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/conf/:/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/jars/*:/home/hadoop/workspace/hadoop-2.7.2/etc/hadoop/' \
org.apache.spark.deploy.SparkSubmit --master yarn --deploy-mode cluster \
--class org.apache.spark.examples.SparkPi --supervise --executor-memory 2G \
--total-executor-cores 3 examples/jars/spark-examples_2.11-2.4.8.jar 1000
```

### 2.2 `SparkSubmit` 类

执行的逻辑：

- 以`proxyUser`的用户身份，执行以下操作；
- `YarnClusterApplication::start()`调用`Client::run()`：

### 2.3 `Client`类

- 在 HDFS 的 `/user/${user}/.sparkStaging/${appId}`下建立目录，用于Yarn的localResource，如下所示；
  - 上传spark配置`__spark_conf__.zip`和应用的jar包到HDFS的路径；
  - 对这两个文件，作为Yarn AM的 local resource；


```shell
$ hdfs dfs -ls /user/hadoop/.sparkStaging/application_1631928041971_0067
-rw-r--r--   2 hadoop supergroup     215195 2021-09-28 11:21 /user/hadoop/.sparkStaging/application_1631928041971_0067/__spark_conf__.zip
-rw-r--r--   2 hadoop supergroup    2017859 2021-09-28 11:21 /user/hadoop/.sparkStaging/application_1631928041971_0067/spark-examples_2.11-2.4.8.jar
```

- `__spark_conf__.zip`目录组织如下：

  - `__hadoop_conf__`中包含`HADOOP_CONF_DIR`和`YARN_CONF_DIR`下的所有文件；

    - 包含`SPARK_CONF_DIR`下的xml文件；

   - 包含CLASSPATH中的`log4j.properties`和`metrics.properties`文件；
  
  
    - 将Hadoop Conf 写到`__spark_hadoop_conf__.xml`；
  
  
    - 将Spark Conf 写到`__spark_conf__.properties`；
  

```shell
drwx------. 2 hadoop hadoop  4096 Sep 26 15:32 __hadoop_conf__
-r-x------. 1 hadoop hadoop 33785 Sep 26 15:32 __spark_conf__.properties
-r-x------. 1 hadoop hadoop 96226 Sep 26 15:32 __spark_hadoop_conf__.xml
-r-x------. 1 hadoop hadoop  1523 Sep 26 15:32 log4j.properties
-r-x------. 1 hadoop hadoop   523 Sep 26 15:32 metrics.properties
```

- 添加的额外的spark的属性
  - `spark.yarn.cache.confArchive`：`__spark_conf__.zip`的路径（作为其它container的配置）；
  - `spark.yarn.cache.filenames`、`spark.yarn.cache.timestamps` 、` spark.yarn.cache.visibilities`、`spark.yarn.cache.types`、`spark.yarn.cache.sizes`：spark自身的jar包，每一个都是Yarn app的localResource。




### 2.4 `ApplicationMaster`类

AM启动命令，类似如下：

- `{{PWD}}`这个是HADOOP内置的变量的使用形式；
- `<LOG_DIR>`是跟应用相关的配置，会被Hadoop自动替换；

- `--properties-file`是最重要的参数，其来源可以见 2.3 节；

```shell
{{JAVA_HOME}}/bin/java -server -Xmx4096m -Djava.io.tmpdir={{PWD}}/tmp \
-Dspark.yarn.app.container.log.dir=<LOG_DIR> org.apache.spark.deploy.yarn.ApplicationMaster \
--class 'org.apache.spark.examples.SparkPi' --jar file:/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.8.jar --arg 1000 \
--properties-file {{PWD}}/__spark_conf__/__spark_conf__.properties 1> <LOG_DIR>/stdout 2> <LOG_DIR>/stderr
```

`--properties-file`可以不指定，那么SparkConf的配置，会从ENV或者Java System Properties中读取，因此可以将文件里的spark配置，作为命令行Java -D参数传入（但是这个可能会导致命令行的长度非常长，具体见  issue [[SPARK-14602\]](https://issues.apache.org/jira/browse/SPARK-14602) 和 解决 https://github.com/apache/spark/pull/12487  ）。

同时，会将`spark.yarn.cache.confArchive`作为Executor的local resource（可以不需要）。



## 3 示例

### 3.1 创建应用，并获取应用ID

```
POST http://rm-http-address:port/ws/v1/cluster/apps/new-application
```

### 3.2 执行应用

以 spark example pi 为例：

#### 3.2.1 Spark自身的jars设置

**AM** 设置，两种方案：

- 通过Yarn local resource，将jar包上传到hdfs上，并在http请求体中配置每一个的jar的local resource（大概226个），或者**先手动合成一个jar包**，再配置（只需HDFS一份）；
- 直接通过CLASSPATH，在Yarn的每个节点都在相同位置安装spark的jars文件，然后在HTTP请求体中的CLASSPATH指定该路径（所有Yarn nodemanager节点一份）；

**Executor**设置，两种方案：

- 通过Yarn local resource，Spark（`spark.yarn.jar`） 是通过先将Spark自身的jar包设置`spark.yarn.cache.filenames | timestamps | visibilities | sizes | types`，放在`__spark_conf__.properties`里，然后解析并配置为Executor的local resource；
- 直接通过CLASSPATH，在`__spark_conf__.properties`里设置`spark.executor.extraClasspath`，路径为spark的jars文件（所有Yarn nodemanager节点一份）

#### 3.2.2 Spark应用程度的jar设置

因为应用程序本身的jar包，是不能预先存在集群上的，且上传到全部节点也不现实（除非有NFS等），因此对于应用程序本身的jar包，需要通过Yarn local resource进行配置，仍然分为AM和Executor两部分的配置。

前提准备：

- 将应用程序jar包，上传到HDFS上，获取到JDFS上该文件的路径、大小、时间戳等信息；

**AM：**

- 在HTTP的请求体中构建，如下形式，其中resource、size、timestamp字段需要按照真实值进行修改：

```json
{
  "key":"__app__.jar",
  "value": {
    "resource": "hdfs://node131:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/spark-examples_2.11-2.4.8.jar",
    "type" : "FILE",
    "visibility": "PRIVATE",
    "size": 2017859,
    "timestamp": 1632823976707
  }
}
```

**Executor**：

- 在`_spark_conf__.properties`，配置`spark.yarn.cache.filenames | timestamps | visibilities | sizes | types`，如下所示

```python
spark.yarn.cache.filenames=hdfs://node131:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/spark-examples_2.11-2.4.8.jar
spark.yarn.cache.timestamps=1632823976707
spark.yarn.cache.visibilities=PRIVATE
spark.yarn.cache.sizes=2017859
spark.yarn.cache.types=FILE
```

#### 3.2.2 Spark配置

1）**构建`__spark_conf__.zip`**

根据 2.3 节内容，可知zip包格式如下：

```python
# hadoop的配置目录
- __hadoop_conf__
  - hdfs-site.xml
  - ...

# spark 配置，核心文件
- __spark_conf__.properties

# hadoop 配置，优先级比__hadoop_conf__高
- _spark_hadoop_conf__.xml
```

`__spark_conf__.zip`作为AM的Yarn local resource，在HTTP请求格式，类似如下：

```json
{
  "key":"__spark_conf__",
  "value": {
    // resource, size, timestamp 需要修改（注意是ARCHIVE类型）
    "resource": "hdfs://node131:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/__spark_conf__.zip",
    "type" : "ARCHIVE",
    "visibility": "APPLICATION",
    "size": 18023,
    "timestamp": 1632825115975
  }
}
```

2）**或 作为命令行/系统配置传递**

`__spark_conf__.properties`中的配置，可以作为HTTP请求体中的command的java 系统配置传入（形式`-Dspark.a.b.c=de`的格式

#### 3.2.3 调用Restful

注：

- 对于Spark 自身的jar包，采用所有节点相同目录部署的形式；
- Spark自身应用程序的jar包，采用Yarn local resource的形式；

Yarn RM接口：`POST http://rm-http-address:port/ws/v1/cluster/apps`

1）**只需要 `__spark_conf__.properties`**

注: 如果只需要`__spark_conf__.properties` 文件，也可以只设置这个文件的local resource，上传到HDFS也可以只上传该文件，而不需要整个`__spark_conf__.zip`

`__spark_conf__.properties`示例：

```python
#Spark configuration.
#Tue Sep 28 17:09:43 CST 2021
spark.executor.memory=2G
spark.cores.max=3
# 需要修改
spark.yarn.cache.confArchive=hdfs\://node131\:9000/tmp/__app__.properties
spark.eventLog.compress=true
spark.submit.deployMode=cluster
spark.ui.killEnabled=false
spark.eventLog.enabled=true
spark.yarn.jars=
spark.yarn.historyServer.address=172.16.2.131\:18080
spark.master=yarn
spark.executor.cores=1
spark.app.name=org.apache.spark.examples.SparkPi
spark.port.maxRetries=100
spark.eventLog.dir=hdfs\://node131\:9000/history
spark.driver.extraClassPath=/extraSparkJars
spark.executor.extraClassPath=/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/jars/*
# filenames, timestamps, sizes 需要修改
spark.yarn.cache.filenames=hdfs\://node131\:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/spark-examples_2.11-2.4.8.jar#__app__.jar
spark.yarn.cache.timestamps=1632823976707
spark.yarn.cache.visibilities=PRIVATE
spark.yarn.cache.sizes=2017859
spark.yarn.cache.types=FILE
```

Yarn Rest 接口的请求体示例：

```json
{
  // 修改应用id
  "application-id": "application_1631928041971_0085",
  "application-name": "Spark_Yarn_Rest",
  "queue": "default",
  "priority": 10,
  "am-container-spec": {
    "local-resources": {
       "entry": [
         {
           "key":"__app__.properties",
           "value": {
             // resource, size, timestamp 需要修改
             "resource": "hdfs://node131:9000/tmp/__spark_conf__.properties",
             "type" : "FILE",
             "visibility": "APPLICATION",
             "size": 990,
             "timestamp": 1632879896059
           }
         }, {
           "key":"__app__.jar",
           "value": {
             // resource, size, timestamp 需要修改
             "resource": "hdfs://node131:9000/tmp/spark-examples_2.11-2.4.8.jar",
             "type" : "FILE",
             "visibility": "PRIVATE",
             "size": 2017859,
             "timestamp": 1632822059125
           }
         }
       ]
    },
    "environment": {
       "entry": [{
            "key": "CLASSPATH",
            "value": "{{PWD}}<CPS>{{PWD}}/__spark_conf__<CPS>/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/jars/*<CPS>$HADOOP_CONF_DIR<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/*<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/lib/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*<CPS>{{PWD}}/__spark_conf__/__hadoop_conf__<CPS>"
          },
          {
            "key": "SPARK_USER",
            "value": "hadoop"
          },
          {
            "key": "PYTHONHASHSEED",
            "value": "0"
          }
        ]
    },
    "commands": {
      "command": "{{JAVA_HOME}}/bin/java -server -Xmx4096m -Djava.io.tmpdir={{PWD}}/tmp -Dspark.yarn.app.container.log.dir=<LOG_DIR> org.apache.spark.deploy.yarn.ApplicationMaster --class 'org.apache.spark.examples.SparkPi' --jar file:/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.8.jar --arg 1000 --properties-file {{PWD}}/__app__.properties 1> <LOG_DIR>/stdout 2> <LOG_DIR>/stderr"
    },
    "service-data": null,
    "credentials": null,
    "application-acls": null
  },
  "unmanaged-AM": false,
  "max-app-attempts": 2,
  "resource": {
     "memory":1024,
     "vCores":1
  },
  "application-type": "Spark",
  "keep-containers-across-application-attempts": true,
  "application-tags": {
    
  },
  "log-aggregation-context": {
    
  },
  "attempt-failures-validity-interval": 3600000,
  "reservation-id": null,
  "am-black-listing-requests": null  
}
```

2）**采用`__spark_conf__.zip`传递配置**

其中有**注释的配置，需要进行修改后，删除注释**，再进行调用：

```json
{
  // 修改应用id
  "application-id": "application_1631928041971_0085",
  "application-name": "Spark_Yarn_Rest",
  "queue": "default",
  "priority": 10,
  "am-container-spec": {
    "local-resources": {
       "entry": [
         {
           "key":"__spark_conf__",
           "value": {
             // resource, size, timestamp 需要修改
             "resource": "hdfs://node131:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/__spark_conf__.zip",
             "type" : "ARCHIVE",
             "visibility": "APPLICATION",
             "size": 18023,
             "timestamp": 1632825115975
           }
         }, {
           "key":"__app__.jar",
           "value": {
             // resource, size, timestamp 需要修改
             "resource": "hdfs://node131:9000/user/hadoop/.sparkStaging/application_1631928041971_0087/spark-examples_2.11-2.4.8.jar",
             "type" : "FILE",
             "visibility": "PRIVATE",
             "size": 2017859,
             "timestamp": 1632823976707
           }
         }
       ]
    },
    "environment": {
       "entry": [{
            "key": "CLASSPATH",
             // resource, size, timestamp 需要修改
            "value": "/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/jars/*<CPS>{{PWD}}<CPS>{{PWD}}/__spark_conf__<CPS>$HADOOP_CONF_DIR<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/*<CPS>$HADOOP_COMMON_HOME/share/hadoop/common/lib/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/*<CPS>$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/*<CPS>$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*<CPS>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*<CPS>{{PWD}}/__spark_conf__/__hadoop_conf__<CPS>"
          },
          {
            "key": "SPARK_USER",
            "value": "hadoop"
          },
          {
            "key": "PYTHONHASHSEED",
            "value": "0"
          }
        ]
    },
    "commands": {
      "command": "{{JAVA_HOME}}/bin/java -server -Xmx4096m -Djava.io.tmpdir={{PWD}}/tmp -Dspark.yarn.app.container.log.dir=<LOG_DIR> org.apache.spark.deploy.yarn.ApplicationMaster --class 'org.apache.spark.examples.SparkPi' --jar file:/home/hadoop/workspace/spark-2.4.8-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.8.jar --arg 1000 --properties-file {{PWD}}/__spark_conf__/__spark_conf__.properties 1> <LOG_DIR>/stdout 2> <LOG_DIR>/stderr"
    },
    "service-data": null,
    "credentials": null,
    "application-acls": null
  },
  "unmanaged-AM": false,
  "max-app-attempts": 2,
  "resource": {
     "memory":1024,
     "vCores":1
  },
  "application-type": "Spark",
  "keep-containers-across-application-attempts": true,
  "application-tags": {
    
  },
  "log-aggregation-context": {
    
  },
  "attempt-failures-validity-interval": 3600000,
  "reservation-id": null,
  "am-black-listing-requests": null  
}
```
