# POSIX ACLs

在 linux下,对一个文件(或者资源)可以进行操作的对象被分为三类: 

- file owner(文件的拥有者)，group(组,注意不一定是文件拥有者所在的组)，other (其他)
- 对于每一类别又分别定义了read, write and execute/search 权限（这里不讨论SUID, SGID以及Sticky bit的设置）。

这种权限控制粒度太粗。ACL就是可以**设置特定用户或者用户组对于一个文件/文件夹的操作权限**。

## 安装设置

```shell
yum -y install libacl acl
```

需要磁盘分区的支持永久启用acl（**centos 7好像不需要以下的操作就可以支持ACL**），以分区/data为例

```shell
# vi /etc/fstab
LABEL=/data /data ext3 defaults,acl 1 2
```

在启用了acl参数之后重新加载/data分区

```shell
mount -o remount /data
cat /etc/mtab | grep /data
# /dev/sda5 /data ext3 rw,acl 0 0
```

出现上面的信息代表分区的acl功能已经正常加载。

## ACL名词解释

ACL 是由一系列的Access Entry所组成的. 每一条Access Entry定义了特定的类别可以对
文件拥有的操作权限. 

Access Entry有三个组成部分: **Entry tag type**, **qualifier(optional)**, **权限**

**Entry tag type**, 它有以下几个类型

- **ACL_USER_OBJ**: 相当于Linux里file_owner的权限
- **ACL_USER**: 定义了额外的用户可以对此文件拥有的权限
- **ACL_GROUP_OBJ**: 相当于Linux里group的权限
- **ACL_GROUP**: 定义了额外的组可以对此文件拥有的权限
- **ACL_MASK**: 定义了**ACL_USER**, **ACL_GROUP_OBJ**和**ACL_GROUP**的最大权限
- **ACL_OTHER**: 相当于Linux里other的权限

**qualifier**，定义了特定用户和用户组，只有user和group才有qualifier。



**`setfacl`**用来设置文件/目录的ACL

```shell
setfacl -m u:zyq:rw test.txt
```

**`getfacl`**命令来查看一个定义好了的ACL文件

```shell
[root@zyq-server data]# getfacl test.txt
# file: test.txt
# owner: root
# group: family
user::rw-          		# 定义了ACL_USER_OBJ, 说明file owner拥有读和写的权限
user:zyq:rw-            # 定义了ACL_USER,这样用户zyq就拥有了对文件的读写权
group::rw-              # 定义了ACL_GROUP_OBJ,说明文件的group拥有read和write 权限
group:jackuser:rw-      # 定义了ACL_GROUP,使得jackuser组拥有了对文件的read 和write权限
mask::rw-               # 定义了ACL_MASK的权限为read and write
other::---              # 定义了ACL_OTHER的没有任何权限操作此文件
```

当任何一个文件拥有了ACL_USER或者ACL_GROUP的值以后可以称它为ACL文件，+号就是用来提示

```shell
[root@zyq-server data]# ll test.txt
-rw-rw-r--+ 1 root root 0 12-27 22:55 test.txt	
```

## ACL_MASK 和 Effective 权限

1） 如果**文件有ACL_MASK值,那么当中那个rw-代表的就是mask值而不再是group 权限**；

2） **ACL_MASK**的规定了ACL_USER, ACL_GROUP_OBJ和ACL_GROUP的最大权限；

##  Default ACL

Default ACL是指对于**一个目录进行Default ACL设置，并且在此目录下建立的文件都将继承此目录的ACL**。

```shell
# d 指 default
[root@zyq-server data]# setfacl -m d:smbuser:rw mydir/
[root@zyq-server data]# getfacl mydir/
# file: mydir/
# owner: root
# group: root
user::rwx
group::r-x
other::r-x
default:user::rwx
default:user:smbuser:rw-
default:group::r-x
default:mask::rwx
default:other::r-x
```

创建新文件或子目录时，它会自动**将其父目录的默认ACL复制到自己的访问ACL**中。

**对父级默认ACL的后续更改不会更改现有子级**。