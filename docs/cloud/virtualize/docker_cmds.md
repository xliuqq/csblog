# 使用

## Dockerfile

> 两种运行模式：建议 ENTRYPOINT 和 CMD 都使用 Exec 模式
>
> - Shell 模式：以`/bin/sh -c "task command"`的方式执行任务命令，即等价于 `["/bin/sh", "-c", "task command"]`
>   - 此时`sh`是父进程，而`task command`是子进程，`sh`接受到的信号，默认无法传给子进程，只能超时强行Kill；
> - Exec 模式：只有一个单独的进程

```shell
# /bin/sh -c "sleep 100"  # 命令所启动的进程关系
$ ps -ef |grep 500109 | grep -v grep
root      500109  500096  0 May21 pts/2    00:00:00 -bash
root     2356429  500109  0 09:26 pts/2    00:00:00 sleep 100
# ps -ef |grep 500109 | grep -v grep # 直接替换当前 shell，退出后当前shell退出；
root      500109  500096  0 May21 pts/2    00:00:00 sleep 100
```

`docker build ` 根据 Dockerfile 构建镜像；

### 指令集

```dockerfile
# scratch是空白镜像
# FROM <image>:<tag>

# MAINTAINER 

# 定义变量，容器运行时无用，类似于定义局部变量，在 docker build 中用 --build-arg <参数名>=<值> 来覆盖
# ARG <key>=<value>

# 环境变量
# ENV <key>=<value>

# 文件拷贝，<src> 支持通配符和多个源
# COPY <src> <dst>

# 所有的文件复制均使用 COPY 指令，仅在需要本地文件自动解压缩的场合使用 ADD
# ADD <src> <dst>

# RUN <command> (shell格式)
# RUN  ["executable", "param1", "param2"] （exec格式，推荐）
# 在前一个命令创建出的镜像基础上创建一个容器，在容器中运行命令，命令结束后提交容器为新镜像

# CMD <command> (shell格式)
# CMD  ["executable", "param1", "param2"] （exec格式，推荐）
# 容器运行时的默认值，配合ENTRYPOINT使用，会被docker run命令时指定的命令参数覆盖

# 表示表示默认程序，docker run 时指定的 CMD 命令会被当做参数传入
# ENTRYPOINT <command> (shell格式)
# ENTRYPOINT  ["executable", "param1", "param2"] （exec格式，推荐）
# shell格式时，会忽略 CMD 和 docker run的命令的参数，运行在 /bin.sh -c 中，非1号进程

# ONBUILD [INSTRUCTION]
# 添加一个将来执行的触发器指令到镜像中。

# 指定当前用户
# USER

# 指定工作目录
# WORKDIR

# 端口映射，一般不在 Dockerfile 中指定，而是 docker 命令行中使用
# EXPOSE

# 挂载卷，一般不在 Dockerfile 中指定，而是 docker 命令行中使用
# VOLUME
```

### 实践

RUN：

- `RUN apt-get update && apt-get install -y foo`应该放一起，避免缓存问题；

- 不要将所有的命令写在一个RUN指令中（除非用于精简镜像大小）；

Expose：

- 不要在Dockerfile中做端口映射，缺乏可移植性；



## 基本命令

### exec

```bash
# 在正在运行的容器中执行命令
$ docker exec [OPTIONS] Container  Commands [Args]
# 交互式
$ docker exec -it [OPTIONS] Container  Commands [Args]  
```

### run

```bash
$ docker run [OPTION] [imageName] [shellCmds]
# 如下，cat /etc/os-release 是执行的CMD，会覆盖DockerFile中的CMD
$ docker run -it ubuntu cat /etc/os-release
$ docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py
```

- **-t** ： 在容器内指定一个伪终端或者终端；  
- **-i** ： 对容器内的标准输入进行交互
- **-d** ： 后台运行，返回唯一的容器ID；     
- **-v**：挂载volume,     [host-dir]:[container-dir]:[rw|ro]
- **-c**：分配容器中所有进程的CPU的share值  
- **-m**：分配容器的内存总量，B、K、M、G单位
- **-P** :是容器内部端口随机映射到主机的高端口；    
- **-p** : 是容器内部端口绑定到指定的主机端口。
- **--rm**：容器运行退出后自动删除
- **--runtime string**：Runtime to use for this container

### images

docker images 来列出本地主机上的镜像

### commit

将正在运行的容器制作新的镜像；

```bash
$ docker commit [OPTIONS] Container [Reopsitory[:TAG]]
```

### build

根据Dockerfile生成镜像

```bash
# 根据当前目录下的Dockerfile生成helloworld镜像
$ docker build -t helloworld .
```

### tag

给容器镜像起完整的名字

```bash
$ docker tag helloworld lzq/helloworld:v1
```

### push

将镜像推送到远程仓库

```
$ docker push lzq/helloworld:v1
```



### rmi

```shell
$ docker rmi f8ab12e03d53
Error response from daemon: conflict: unable to delete f8ab12e03d53 (must be forced) - image is referenced in multiple repositories
```

删除时可以用**repository和tag的方式**来删除

```bash
$ docker rmi 192.168.0.1/you/tom:1.0.8
Untagged:192.168.0.1/you/tom:1.0.8
```



### export/save

`docker export`：将容器的文件系统导出为`tar`，加载为`docker import`

- **容器**持久化（单层Linux文件系统），丢失历史，加载时可以指定镜像名称；
- 适合基础镜像；

`docker save`：镜像导出为 `tar`，加载为`docker load`

- **镜像**持久化（分层文件系统），不丢失历史和层，可通过`docker tag`命令实现层回滚；
- 加载时不可指定镜像名称；

