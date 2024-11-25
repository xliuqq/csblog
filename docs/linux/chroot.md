# Chroot

chroot，即 change root directory (更改 root 目录)。在 linux 系统中，系统默认的目录结构都是以 /，即以根 (root) 开始的。而在使用 chroot 之后，系统的目录结构将以指定的位置作为 / 位置。

- **增加了系统的安全性，限制了用户的权力：**
- **建立一个与原系统隔离的系统目录结构，方便用户的开发：**
- **切换系统的根目录位置，引导 Linux 系统启动以及急救系统等：**



## 示例使用

busybox 包含了丰富的工具，我们可以把这些工具放置在一个目录下，然后通过 chroot 构造出一个 mini 系统。

```shell
# 在当前目录下创建一个目录 rootfs
$ mkdir rootfs
# 把 busybox 镜像中的文件解压到 rootfs 目录
$ (docker export $(docker create busybox) | tar -C rootfs -xvf -)
# 查看rootfs目录下内容（包括基本的/bin,/dev,/etc等目录）
$ ls rootfs
# 通过chroot执行命令
$ sudo chroot rootfs /bin/pwd
$ sudo chroot rootfs /bin/sh

```

## 原理

等价于调用函数`chroot()`和`chdir()`。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
 
int main(int argc, char *argv[])
{
  if(argc<2){
    printf("Usage: chroot NEWROOT [COMMAND...] \n");
    return 1;
  }
  if(chroot(argv[1])) {
    perror("chroot");
    return 1;
  }
 
  if(chdir("/")) {
    perror("chdir");
    return 1;
  }
 
  if(argc == 2) {
    // hardcode /bin/sh for my busybox tools.
    argv[0] = (char *)"/bin/sh";
 
    argv[1] = (char *) "-i";
    argv[2] = NULL;
  } else {
    argv += 2;
  }
 
  execvp (argv[0], argv);
  printf("chroot: cannot run command `%s`\n", *argv);
 
  return 0;
}
```

