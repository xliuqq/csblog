# 常用镜像

## Alpine

> musl libc 和 BusyBox.

### Alpine-docker容器中安装GCC

```shell
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories

RUN apt add --update  build-base && rm -rf /var/cache/apk/*

```

`build-base` is a meta-package that will install the GCC, libc-dev and binutils packages (amongst others).



