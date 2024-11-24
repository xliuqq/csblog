# 使用

## 部署

```bash
# yum更新
yum update

# 安装必备系统软件
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加repo
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 更新yum缓存
yum makecache fast

# 安装docker
yum -y install docker-ce

# 启动 Docker 后台服务
systemctl start docker
systemctl enable docker

# 测试运行 hello-world
docker run hello-world
```



## 源修改

修改或新增 `/etc/docker/daemon.json`

```text
{
"registry-mirrors": "https://docker.linkedbus.com"
}
```

然后执行：

```bash
systemctl daemon-reload
systemctl restart docker
```



## FAQ

### No route to host

#### 问题描述

在centos部署docker镜像时，遇到**docker容器无法访问宿主机ip**的问题(用的wget工具)，报错信息为：No route to host。而从docker容器中访问宿主机所在局域网的其他主机的服务是可以访问的。

#### 原因

docker容器，其默认网络模式采用的是bridger模式。

启动docker时，docker进程会创建一个名为docker0的虚拟网桥，用于宿主机与容器之间的通信。当启动一个docker容器时，docker容器将会附加到虚拟网桥上，容器内的报文通过docker0向外转发。

如果docker容器**访问宿主机**，那么docker0网桥将报文直接转发到本机，**报文的源地址是docker0网段的地址**。

而如果docker容器**访问宿主机以外的机器**，docker的SNAT网桥会将报文的源地址**转换为宿主机的地址**，通过宿主机的网卡向外发送。

因此，**当docker容器访问宿主机时，宿主机服务端口会被防火墙拦截，从而无法连通宿主机，出现No route to host的错误**。而访问宿主机所在局域网内的其他机器，由于报文的源地址是宿主机ip，因此，不会被目的机器防火墙拦截，所以可以访问。

#### 解决方案

**方法一：关闭防火墙**

centos关闭防火墙的操作为

`systemctl stop firewalld`

**方法二： 在防火墙上开发指定端口**

```shell
firewall-cmd --zone=public --add-port=8090/tcp --permanent
firewall-cmd --reload
```