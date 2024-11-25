# Bash

GNU Bourne-Again SHell

**login shell** ：登录的shell

**interactive shell** ：执行脚本会变成 non-interactive shell

## 登录方式

<img src=".pics/bash/BashStartupFiles.png" alt="Bash加载文件顺序" style="zoom: 90%;" />

### interactive + login shell

`bash -l`命令，它会打开一个login shell；

`ssh user@remote` 也会得到一个login shell；

shell首先加载`/etc/profile`，然后再尝试依次去加载下列三个配置文件之一，**一旦找到其中一个便不再接着寻找**：

- **~/.bash_profile**
- **~/.bash_login**
- **~/.profile**

`.bash_profile`和`.profile`可以很好的处理bash和Bourne shell之间的切换。

最佳实践：

- 应该尽量杜绝使用`.bash_login`，如果已经创建，那么需要创建`.bash_profile`来屏蔽它被调用
- `.bash_profile`适合放置bash的专属命令，可以在其最后读取`.profile`，如此一来，便可以很好的在Bourne shell和bash之间切换了；



### non-interactive + login shell

`bash -l script.sh`

-l参数是将shell作为一个login shell启动，而执行脚本又使它为non-interactive shell。

**配置文件的加载与第一种完全一样**



### interactive + non-login shell

`bash` 

在一个已有shell中直接运行`bash`，此时会打开一个交互式的shell，而因为不再需要登陆，因此不是login shell。

**启动shell时会去查找并加载`~/.bashrc`文件。**

#### bashrc vs profile

- profile类型文件是某个用户唯一的用来设置全局环境变量的地方，用户可以有多个shell比如bash, sh, zsh等，启动一个login shell会加载此文件，后面由此shell中启动的新shell进程如bash，sh，zsh等都可以由login shell中继承环境变量等配置。
- bashrc，其后缀`rc`的意思为[Run Commands](http://en.wikipedia.org/wiki/Run_commands)，此处存放bash需要运行的命令，但注意，这些命令一般只用于交互式的shell，通常在这里会设置交互所需要的所有信息，比如bash的补全、alias、颜色、提示符等等。



### non-interactive + non-login shell

`bash script.sh`

**找环境变量`BASH_ENV`，将变量的值作为文件名进行查找，如果找到便加载它。**



## ssh 说明

`ssh user@remote cmd`

> ssh man page
>
> If command is specified, it is executed on the remote host instead of a login shell.
>
> When the user's identity has been accepted by the server, the server executes the given command in a non-interactive session.

**即ssh 应该是non-interactive + non-login**，但是bash中有说明

>  Bash attempts to determine when it is being run with its standard input connected to a network connection, as when executed by the **remote shell daemon**, usually rshd, or the  secure shell  daemon sshd.  If bash determines it is being run in this fashion, **it reads and executes commands from ~/.bashrc**, if that file exists and is readable.  It will not do this if invoked as sh. 

即 ssh user@remote cmd 时会加载 ~/.bashrc。

默认Debian bashrc框架中的解决方法是将以下内容放在.bashrc的顶部：

```shell
# If not running interactively, don't do anything
[ -z "$PS1" ] && return
```

远程执行脚本时，进行登录，读取/etc/profile配置。`ssh user@remote bash -l script.sh`



**ssh connection refused**

```shell
# 先查看ssh进程是否存在，不存在则启动
ps -ef | grep ssh
sudo /etc/init.d/ssh start
```



## Sh 软链接

sh 是 bash的软链接。

当bash以是sh命启动时，即我们此处的情况，bash会尽可能的模仿sh，所以配置文件的加载变成了下面这样：

- interactive + login: 读取`/etc/profile`和`~/.profile`
- non-interactive + login: 同上
- interactive + non-login: 读取`ENV`环境变量对应的文件
- non-interactive + non-login: 不读取任何文件