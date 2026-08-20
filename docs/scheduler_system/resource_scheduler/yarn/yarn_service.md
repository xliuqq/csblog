# Yarn Service

> 一个容器编排平台，用于管理YARN上的容器化服务。它既支持docker容器，也支持传统的基于进程的容器。
>
> https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/yarn-service/YarnServiceAPI.html#Examples

- 执行配置解析和挂载；
- 生命周期管理(如停止/启动/删除服务)；
- 伸缩服务组件；
- 在YARN上滚动升级服务；
- 监控服务的健康和就绪情况等等。



提供 Json 配置文件，来运行Long Time Service，不需要写复杂的AM。

1. 一个**核心框架（AM）扮演容器编排**，**负责所有的服务的生命周期管理**。
2. **RESTful服务器用于用户交互 ，部署、管理服务**。
3. **DNS服务依赖YARN Service Registry允许发现服务**。



## 配置

`yarn-site.xml`

```xml
    <property>
        <description>
            Enable services rest api on ResourceManager.
        </description>
        <name>yarn.webapp.api-service.enable</name>
        <value>true</value>
    </property>
```

## CLI

```shell
# 启动一个Service，对于当前用户，SERVICE_NAME必须唯一
$ yarn app -launch ${SERVICE_NAME} ${PATH_TO_SERVICE_DEFINE_FILE}
# 对Service的组件进行扩充/缩减，支持绝对值和相对值(+2, -2)
$ yarn app -flex ${SERVICE_NAME} -component ${COMPONENT_NAME} ${NUMBER_OF_CONTAINERS}
# 停止服务
$ yarn app -stop ${SERVICE_NAME}
# 重启一个停掉的服务
$ yarn app -start ${SERVICE_NAME}
# 删掉服务（会删除HDFS该服务的目录以及Yarn Service Registry记录
$ yarn app -destroy ${SERVICE_NAME}

```

## Restful API

见官网 http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/yarn-service/QuickStart.html#Manage_services_on_YARN_via_REST_API



## 服务注册（Service Registry）TODO



## 服务发现（Service Discovery)

DNS Server 的 DNS lookup 实现 Yarn 上的服务发现，比如HBase集群部署在Yarn上时的master-region通信。

- container分配的节点是随机的，在创建时无法指定master/region在哪个节点，因此需要一种规范机制。

框架的AM会**将Container的信息（如hostname和ip）注册到Yarn Service Registry**，DNS Server将Yarn Service Registry中的信息转换成DNS record，Client就可以通过标准的DNS查询发现container的ip信息。

- 对于非Docker容器，支持**forward DNS lookup**，不支持 reverse DNS lookup；
- 对于Docker容器，支持forward DNS lookup 和 reverse DNS lookup；
- 此外，DNS支持配置 static zone files 进行 forward和reverse lookup；

### 配置（TODO-Registry DNS）



## 系统服务

系统服务是管理员配置的服务，在ResourceManager的bootstrap过程中自动部署。

```xml
    <property>
        <name>yarn.service.system-service.dir</name>
        <value>true</value>
    </property>
```

YarnFiles配置文件存储：`${yarn.service.system-service.dir}/<Launch-Mode>/<Users>/<Yarnfiles>`

- Launch-Mode 分为 `sync`和`async`；
- Users 指的操作系统用户；
- Yarnfiles指的是配置文件，必须是以`.yarnfile`结尾；



## 使用

### 配置文件

`sleeper.json`

```json
{
  "name": "sleeper-service",
  "version": "1.0",
  "components" : 
    [
      {
        "name": "sleeper",
        "number_of_containers": 1,
        "launch_command": "sleep 900000",
        "resource": {
          "cpus": 1, 
          "memory": "256"
       },
        "configuration": {
          "properties": {
            "yarn.service.container-health-threshold.percent": "90",
            "yarn.service.container-health-threshold.window-secs": "400",
            "yarn.service.container-health-threshold.init-delay-secs": "800"
          }
        }
      }
    ]
}
```

启动

```shell
# launch a sleeper service named as my-sleeper on YARN.
$ yarn app -launch my-sleeper sleeper
```

