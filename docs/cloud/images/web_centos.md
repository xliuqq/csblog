# 基于浏览器使用CentOS（多用户）

# 0. noVNC定制化代码部分

- noVNC版本：[https://github.com/novnc/noVNC/tree/v1.2.0](https://github.com/novnc/noVNC/tree/v1.0.0)
- websockify版本：https://github.com/novnc/websockify

## 0.1 设置不同用户url路径不同

- 新增脚本：`/noVNC/app/sed.sh`

- - `WEBSOCK_PATH`参数需**在启动镜像的k8s.yaml中配置，为不同用户不同url路径**
  - 1920x1080为分辨率，根据用户机的显示器大小自行调整

```bash
#!/bin/bash

sed -i "s#WEBSOCK_PATH#${WEBSOCK_PATH}#g" /noVNC/app/ui.js
vncserver -geometry 1920x1080
nohup /noVNC/./utils/launch.sh --vnc 0.0.0.0:5901 1>/dev/null 2>&1 
```

- 修改文件：`/noVNC/app/ui.js`

- - 将注释替换成下一行

```bash
# UI.initSetting('path', 'websockify');
UI.initSetting('path', 'WEBSOCK_PATH/websockify');
```

- 修改文件：`/noVNC/vnc.html`，用于nginx配置转发

- - 将注释替换成下一行

```bash
#<input id="noVNC_setting_path" type="text" value="websockify">
<input id="noVNC_setting_path" type="text" value="/noVNC/websockify">
```

## 0.2 免密登录修改

- 修改文件：/noVNC/app/ui.js

- - 新增部分：`.then(() => {document.getElementById("noVNC_connect_button").click()});`

- - 新增部分：`password = "rootroot";`

```javascript
prime() {
  return WebUtil.initSettings().then(() => {
    if (document.readyState === "interactive" || document.readyState === "complete") {
      return UI.start();
    }
    
    return new Promise((resolve, reject) => {
      document.addEventListener('DOMContentLoaded', () => UI.start().then(resolve).catch(reject));
    });
  }).then(() => {
    document.getElementById("noVNC_connect_button").click()
    });
 },

    .......
      
    password = "rootroot";
    UI.hideStatus();
  
```



# 1. 通过Dockerfile构建可视化终端镜像

## 1.1 文件目录组织

- **Dockerfile**：用于构建可视化终端镜像

- **centos7_vnc_install.sh**：用于安装centos系统相关组件等
- **noVNC**：用于提供vnc代理服务并基于开源1.2.0版本定制了部分内容，详细改动见第0节
- **setvncpasswd.sh**：用于自动化设置vnc密码
- **vncservers**：用于启动vnc服务
- **xstartup**：用于启动xfce远程可视化桌面服务

![novnc_dirs.png](.pics/web_centos/novnc_dirs.png)

## 1.2 Dockerfile文件

```dockerfile
# 基于centos7镜像构建本服务，且该镜像的yum源已配置为国内
FROM centos:7.9.2009
ENV noVNCVersion v1.2
ENV AUTHOR HC
ENV LANG en_US.UTF-8
ENV TZ=Asia/Shanghai
ENV SHELL=/bin/bash
LABEL maintainer="test"
# 设置工作目录
WORKDIR /noVNCSet
COPY . /noVNCSet

# 设置时区，设置root密码，启动centos7_vnc_install.sh安装脚本，启动setvncpasswd.sh密码设置脚本，拷贝xfce配置文件xstartup到vnc目录
RUN set -ex && ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo '$TZ' > /etc/timezone && mv /noVNCSet/noVNC /noVNC && echo -e "root\nroot" | passwd && sh /noVNCSet/centos7_vnc_install.sh && chmod 755 /noVNCSet/setvncpasswd.sh &&  cd /noVNCSet &&  ./setvncpasswd.sh && mv /noVNCSet/xstartup /root/.vnc/xstartup && chmod 755 /root/.vnc/xstartup

EXPOSE 6901

ENTRYPOINT sh /noVNC/app/sed.sh
```

## 1.3 centos7_vnc_install.sh文件

```bash
#/bin/bash

function install_vnc
{
        # 安装wget及centos源管理工具
        yum install -y  wget
        wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
        rpm -ivh epel-release-latest-7.noarch.rpm
        yum update
        yum install deltarpm -y
        # 安装vnc相关服务
        yum install -y "tigervnc-server"
        yum install tcl-devel -y、
        # 安装自动化执行命令服务expect
        yum install expect -y
        # 安装xfce
        yum install epel-release
        yum  -y --enablerepo=epel groupinstall xfce
        # 安装中文字体
        yum -y  --enablerepo=epel groupinstall Fonts
        # 安装火狐浏览器
        yum -y install firefox
        yum -y install python3
        pip3 install numpy -i https://pypi.douban.com/simple/
}

install_vnc
```

## 1.4 noVNC文件（属于定制版本，详细改变看第0节）



## 1.5 setvncpasswd.sh文件

```bash
#!/usr/bin/expect
set timeout 10
spawn vncpasswd
expect "Password:"
send "rootroot\n"
expect "Verify:"
send "rootroot\n"
expect "Would you like to enter a view-only password (y/n)?"
send "n\n"
interact
```

## 1.6 vncservers文件

```bash
# THIS FILE HAS BEEN REPLACED BY /lib/systemd/system/vncserver@.service
VNCSERVERS="1:root"
VNCSERVERARGS[1]="-geometry 1920x1080"
```

## 1.7 xstartup文件

```bash
#!/bin/sh

unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
#/etc/X11/xinit/xinitrc
# Assume either Gnome or KDE will be started by default when installed
# We want to kill the session automatically in this case when user logs out. In case you modify
# /etc/X11/xinit/Xclients or ~/.Xclients yourself to achieve a different result, then you should
# be responsible to modify below code to avoid that your session will be automatically killed
if [ -e /usr/bin/gnome-session -o -e /usr/bin/startkde ]; then
vncserver -kill $DISPLAY
fi
```

# 2. 将noVNC服务启动命令加到k8s的yaml中

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hc-terminal-pod
  namespace: ai-education
  labels:
    app: hc-terminal-pod-label
spec:
  containers:
  - name: hc-terminal-container1
    securityContext:
      privileged: true
    image: 172.16.2.131:30002/starfish/hc-terminal:v3.0               # 第1节通过dockerfile构建出来的镜像
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        memory: 100Mi
      limits:
        memory: 2000Mi
      requests:
        cpu: "1"
      limits:
        cpu: "1"
    ports:
    - name: http
      containerPort: 6901
    env:
      - name: WEBSOCK_PATH
        value: "001"
    command: ["/bin/bash"]                        #容器启动命令，进入容器的shell终端
    args: ["-c","su root -s /bin/bash  /noVNC/app/sed.sh"]  #启动noVnc
---
apiVersion: v1
kind: Service
metadata:
  name: hc-terminal-service
  namespace: ai-education
spec:
  ports:
  - name: http
    port: 6901                           #terminal对外暴露服务的端口号
    protocol: TCP
  selector:
    app: hc-terminal-pod-label
  type: NodePort
```