# Systemctl

## Service

service命令其实是去/etc/init.d目录下，去执行相关程序

```shell
# service命令启动redis脚本
service redis start
# 直接启动redis脚本
/etc/init.d/redis start
# 开机自启动
update-rc.d redis defaults
```

## Systemctl

`systemd`是Linux系统最新的初始化系统(init)，作用是提高系统的启动速度，尽可能启动较少的进程，尽可能更多进程并发启动。

`systemd`对应的进程管理命令是`systemctl`。

1. **systemctl命令兼容了service**
   - 即 `systemctl` 也会去`/etc/init.d`目录下，查看，执行相关程序
2. **systemctl命令管理systemd的资源Unit**

命令形式：

```shell
$ systemctl [command] [unit]
```

command 主要有：

- start：立刻启动后面接的 unit。
- stop：立刻关闭后面接的 unit。
- restart：立刻关闭后启动后面接的 unit，亦即执行 stop 再 start 的意思。
- reload：不关闭 unit 的情况下，重新载入配置文件，让设置生效。
- enable：设置下次开机时，后面接的 unit 会被启动。
- disable：设置下次开机时，后面接的 unit 不会被启动。
- status：目前后面接的这个 unit 的状态，会列出有没有正在执行、开机时是否启动等信息。
- is-active：目前有没有正在运行中。
- is-enable：开机时有没有默认要启用这个 unit。
- kill ：不要被 kill 这个名字吓着了，它其实是向运行 unit 的进程发送信号。
- show：列出 unit 的配置。
- mask：注销 unit，注销后你就无法启动这个 unit 了。
- unmask：取消对 unit 的注销。

### .mount

[Mount] 节点里配置了 What,Where,Type，等同于以下命令：

```shell
# What=hugentlbfs WHere=/dev/hugepages Type=hugetlbfs
mount -t hugetlbfs /dev/hugepages hugetlbfs
```

### .service

```ini
[Unit]
# 描述
Description=nginx
# 依赖的服务
After=network.target

[Service]
# 如果服务程序启动后会通过 fork 系统调用创建子进程，然后关闭应用程序本身进程的情况，则应该将 Type 的值设置为 forking
Type=forking 
ExecStart=/usr/local/nginx/sbin/nginx 
ExecReload=/usr/local/nginx/sbin/nginx -s reload 
ExecStop=/usr/local/nginx/sbin/nginx -s quit
# 指定在什么情况下需要重启服务进程。常用的值有 no，on-success，on-failure，on-abnormal，on-abort 和 always
Restart=no
# 指定运行服务的用户，会影响服务对本地文件系统的访问权限。
User=dev
# 是否给服务分配独立的临时空间（true/false）
PrivateTmp=true 

[Install] 
# 当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。
WantedBy=multi-user.target
```

### .target

定义了一些基础的组件，供.service文件调用

### .wants

.wants文件定义了要执行的文件集合，每次执行，.wants文件夹里面的文件都会执行

