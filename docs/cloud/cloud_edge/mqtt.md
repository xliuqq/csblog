# MQTT

## MQTT协议

MQTT（**Message Queuing Telemetry Transport**）协议 是ISO 标准下的一种基于**发布 - 订阅**模式的消息协议，**基于 TCP/IP 协议簇**。

其适用于**物联网**消息传递，例如低功率传感器或移动设备，如电话，嵌入式计算机或微控制器。

![publish-subscribe-arch](.pics/mqtt/mqtt-publish-subscribe.png)

**Apache Pulsar 原生支持 MQTT。**



### MQTT-SN

SN：Sensor Networks，MQTT-SN是MQTT协议的传感器版本。



## MQTT Client 实现

### [Paho](https://www.eclipse.org/paho/)

[MQTT 不同 Client 的 对比](https://www.eclipse.org/paho/index.php?page=downloads.php)



## MQTT Broker 实现

完整的列表见[这里](https://github.com/mqtt/mqtt.org/wiki/servers)

### [EMQX](https://github.com/emqx/emqx)(Erlang，APL 2.0)

- 完整 MQTT 3.x 和 5.0 规范
- Masterless 高可用集群架构
- 高并发、低时延、高性能
- 可扩展的网关和插件体系
- **数据存储是企业版的功能**



### [Eclipse Mosquitto](http://mosquitto.org/)(C，Apache 2.0)

>  An open source MQTT broker. https://github.com/eclipse/mosquitto

一个开源（EPL / EDL许可）消息代理，它实现了MQTT协议版本 5.0，3.1和3.1.1。

**重量轻**，适用于从**低功耗单板计算机**到完整服务器的所有设备。

提供 C library Client 和 CLI。

