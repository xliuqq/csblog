# Linux系统中的日志管理

`/run`：是一个**临时文件系统**（[tmpfs](./fs.md#ramdisk)），存储系统启动以来的信息。当系统重启时，这个目录下的文件应该被删掉或清除。



## Journald

收集来自内核、系统早期启动阶段的日志、**系统守护进程**在启动和运行中的标准输出和错误信息，还有 **syslog** 的日志。

journald日志写入**二进制文件**。

存储位置：

- `volatile`表示仅保存在内存中， 也就是仅保存在 `/run/log/journal` 目录中(将会被自动按需创建)；
-  `persistent`表示优先保存在磁盘上， 也就优先保存在 `/var/log/journal` 目录中(将会被自动按需创建)；
- "`auto`"(默认值) 与 "`persistent`" 类似， 但不自动创建 `/var/log/journal` 目录， 因此可以根据该目录的存在与否决定日志的保存位置。

**会根据文件大小，设置滚动策略，以及总的文件大小，进行清除策略**。



## rsyslog 

rsyslog作为传统的系统日志服务，把所有收集到的日志都记录到`/var/log/`目录下的各个日志文件中。



## logrorate

> Linux写文件是根据inode编号，**重命名文件不影响文件的读写**；
>
> - 默认，创建一个新的日志文件给程序输出日志，通过某种机制通知程序，如 nginx 支持 kill -USR1，<font color='red'>**需要程序支持**</font>；
> - 配合cron使用，如果 logrotate 配置了daily，则一天内即使cron重复调用，也不会输出两份备份；

日志文件管理工具，**删除旧日志**并**创建新的日志**，以及**压缩日志**和发送日志到email；

```shell
$ logrorate -s <statefile> <conffile>
```

- 使用 `logrotate -d /etc/logrotate.d/myapp` 命令来测试你的配置文件是否有效，而不实际执行轮转操作

```
# 在`/etc/logrotate.d/`目录中新增单独的配置文件（.d不是默认 daily 的意思），会自动执行
# /etc/logrotate.d/myapp

/var/log/myapp/myapp.log {  # 路径支持正则匹配
    daily               # 每天轮转一次  
    rotate 7            # 保留最近 7 个轮转后的备份文件  
    compress            # 对轮转后的备份文件进行压缩  
    delaycompress       # 延迟压缩，直到下一次轮转时再进行压缩  
    missingok           # 如果日志文件不存在，则忽略错误继续  
    notifempty          # 如果日志文件为空，则不轮转  
    create 640 root adm # 创建新的日志文件，权限为 640，所有者为 root，组为 adm  
    sharedscripts       # 如果多个日志文件匹配此规则，则只运行一次 postrotate/endscript 脚本  
  
    # 在轮转后执行的命令  
    postrotate  
        /usr/bin/systemctl reload myapp  # 假设 myapp 是一个 systemd 服务，重新加载它以应用新的日志文件  
    endscript  
}
```

参数说明：

- compress | nocompress：gzip压缩转储后的日志；

- copytruncate      | nocopytruncate ： 打开中的日志文件，当前日志备份并截断（丢失日志的风险）；

- ifempty | notifemply：空文件也转储（默认）；

- rotate N ：日志文件删除前的转储次数，0指没有备份，5指保留5个备份；

- create mode owner group：转储文件，使用指定的文件模式创建新的日志文件；

- prerotate/endscript：转储前需要执行的命令，关键字单独成行；

- postrotate/endscript：转储后需要执行的命令，关键字单独成行；

- missingok |nomissingok：日志文件不存在，继续下一个而不报错；

- daily | weekly |monthly：转储周期；

- size=19：日志文件到指定大小再转储，默认bytes，可选K和M；

