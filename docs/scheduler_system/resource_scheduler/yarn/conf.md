# 配置

## 基本配置



## GPU基本配置

见 [GPU](./gpu.md)

## yarn

### yarn.scheduler.capacity.resource-calculator

`org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator`：默认

`org.apache.hadoop.yarn.util.resource.DominantResourceCalculator`：一般需要改成这个，web ui显示的和实际申请的会一致。



## 日志聚合

`yarn-site.xml`

- 远程的日志路径是 `{yarn.nodemanager.remote-app-log-dir}/${user}/{yarn.nodemanager.remote-app-log-dir-suffix}`

```xml
<property>
    <!-- 下面路径是默认值，不需要配置-->
	<name>yarn.log.server.url</name>
	<value>http://node01:19888/jobhistory/logs</value>
</property>

<property>
    <!-- 默认不开启 -->
    <name>yarn.log-aggregation-enable</name>  
    <value>true</value>  
</property>  
<property>  
    <!-- 设置日志保留时间，单位是秒，默认 -1，不会删除聚合后的日志 -->
    <name>yarn.log-aggregation.retain-seconds</name>  
    <value>640800</value>  
</property>
<property>  
    <!-- 删除任务在HDFS上执行的间隔，执行时候将满足条件的日志删除（超过参数“yarn.log-aggregation.retain-seconds”设置的时间的日志），默认 -1，不检查 -->
    <name>yarn.log-aggregation.retain-check-interval-seconds</name>  
    <value>640800</value>  
</property>
<property>  
    <!-- 当应用程序运行结束后，日志被转移到的HDFS目录（启用日志聚集功能时有效），修改为保存的日志文件夹。默认是 /tmp/logs -->
    <name>yarn.nodemanager.remote-app-log-dir</name>  
    <value>/hdfs/path/to/logs</value>  
</property>
<property>
    <!-- 为每个应用程序的日志目录添加一个后缀。可以更好地组织和管理日志 -->
    <!-- logs 日志将被转移到目录`{yarn.nodemanager.remote−app−log−dir}/{user}/${thisParam}`下-->
    <name>yarn.nodemanager.remote-app-log-dir-suffix</name>  
    <value>logs-${user.name}</value>  
</property>
<property>
    <!-- aggregated log files per application per NM -->
    <name>yarn.nodemanager.log-aggregation.num-log-files-per-app</name>
    <value>30</value>
</property>

<property>
    <!-- 默认情况下，日志将在应用程序完成时上传。通过设置此配置，可以在应用程序运行时定期上传日志。默认 -1 -->
    <name>yarn.nodemanager.log-aggregation.roll-monitoring-interval-seconds</name>
    <value>3600</value>
</property>
```


