# Shell 编程

I/O标识为标准输入0，标准输出1，标准错误2； > 重定向输出， >> 附加内容；

## bash，source，exec

- bash执行shell脚本，产生**sub shell**（即fork，执行产生新的子进程，执行完后返回到父进程）执行，执行完后不保留子进程的信息；
- **source执行shell脚本，在当前shell环境中执行**；
- exec 执行不产生新的子进程，但是会替换当前的shell，执行完关闭（一般将exec放到shell脚本中），即**exec后面的脚本不会执行**；



sh 参数： **-n 只检查语法错误**，不执行脚本； **-v 运行命令前先显示命令**； **-x 处理命令后显示命令**

- **-e** ：如果某个指令的执行返回非0，则直接返回，不需要通过 `$?`判断返回值
  - 注意跟 grep 的配合（grep匹配不到则返回非0），可以用 `$(ps -ef | { grep "进程标识" || true; })`

## `${}, $()`

在bash中，`$( )`与\` \`（反引号）都是用来作命令替换的。

`$var`与`${var}`是没有区别的，但是用`${ }`会比较精确的界定变量名称的范围

``` shell
PKG_DIR=$(ls)
for apiName in ${PKG_DIR[*]} ; do echo $apiName; done
for apiName in ${PKG_DIR} ; do echo $apiName; done
```



## 基本语法

### Set

```shell
# 表示一旦脚本中有命令的返回值为非0，则脚本立即退出，后续命令不再执行;
set -e 
# 以调试的形式运行，即会打印出每次执行的语句
set -x
# 表示在管道连接的命令序列中，只要有任何一个命令返回非0值，则整个管道返回非0值，即使最后一个命令返回0.
set -o pipefail 
```



### 变量

``` bash
A="hello world"
```

- 等号两边不能有空格；
- `readonly`用于定义只读变量；

### for

``` shell
for var in value_list ; do
      ...
done
```

注意：

```bash
for line in `cat file`; do done
```

用法，**file 中的空格会被当成行分隔符**，需要指定 `IFS=$'\n'` 才可以正常分行

### while

> read line 读取时，如果最后一行没有换行符则无法读取，可以通过以下形式
>
> ```
> while read line || [ -n $line ]
> do
> done < a.txt
> ```

``` shell
while true
do
    command
done

while true; do $command; done;

# 等价于
for (( ; ; ))
```

### if

``` shell
if condition_judgement ; then 
... 
elif condition2; then 
... 
else 
... 
fi
```

### case

``` shell
case experssion in
pattern1)
      ...;;
pattern2)
      ...;;
*)
   ...;;
esac
```

### 数组

``` shell
array_name=(value1 ... valuen)
```

### 参数

- `$?` : 前一个命令的退出状态，正常0，异常非0；
- `$#` : 脚本或函数的参数个数； $$ : Shell本身的PID
- `$0` : 脚本名称； `$1, $2 … `： 脚本或函数的位置参数；
- `$*` : 以 `$1 $2 …` 保存用户传入的位置参数；` $@` : 以 `$1` `$2 …`$n` 形式保存位置参数

### 运算

```shell
# 整形运算： 
((a=$j+$k))  #或 
let a+= 1 #或 
a=`expr $a + 1` #（加号两边有空格），
((i=$j*$k)) #等价 
i=`expr $j \* $k`  
((i=$j\$k))  #等价
i=`expr $j /$ $k`  

echo $(($j + $k)

```

### 函数

```shell
# 先定义，再使用（无括号，参数加空格）  function_name arg_a arg_b
function_name() { 
       ...	# 函数参数同脚本，通过$1，$2表示
}	
# 函数结尾可以用return语句返回执行状态，0表示无错误；
# 函数默认是将标准输出传递出来，不是返回值；或者使用全局变量
```



## 比较

test用来比较数字、字符串、文件状态检查、逻辑判()断等，test condition 或 [ condition ]，两边有空格，条件成立返回0，否则非0

- 字符串： == 或= , != , -z (空), -n(非空) ；
- 整型： -eq , -ne , -gt , -ge , -lt , -le ；
- 逻辑与（-a)或（-o）非（!）；
- 在[]中使用==是bash里的做法, 不符合posix标准；
- =~ 正则表达式匹配
- ${} 变量，或者执行命令；



文件相关判断：

- `-a`：FILE是文件存在则为真；
- `-d`：FILE是目录存在则为真；
- `-e`：FILE存在则为真；



## 参数获取

关于参数的内置变量

- **$#** : 脚本或函数的参数个数；
- **$$** : Shell本身的PID
- **$0** : 脚本名称； **$1, $2** … ： 脚本或函数的位置参数；



对于命名的参数获取：**getopt**

getopt命令不是一个标准的unix命令，但它在大多数Linux的发行版中都自带了有，如果没有，也可以从[getopt官网](http://software.frodo.looijaard.name/getopt/)上下载安装。

```shell
ARGS=`getopt -o ab:c:: --long along,blong:,clong:: -n 'example.sh' -- "$@"`
```

- **-o或--options**选项后面接可接受的短选项，如ab:c::，表示可接受的短选项为-a -b -c，其中-a选项不接参数，-b选项后必须接参数，-c选项的参数为可选的；

- **-l**或**--long**选项后面接可接受的长选项，用逗号分开，冒号的意义同短选项。



示例：

```bash
# /bin/bash

show_usage(){
    msg="Generate git two commits diff files, echo modified file with a diff file.\n\
usage: \n\
-b,--begin-commit-id: begin commit id(inclusive) \n\
-e,--end-comiti-d: end commit id(inclusive) \n\
-h,--help"
    echo -e $msg
}

# 解析命令
ARGS=`getopt -o b:e:h -l begin-commit-id:,end-comiti-id:,help -- "$@"`
if [ $? != 0 ]; then
    echo "Error: args is not valid, use -h to show usage help!"
    exit -1
fi

# 将规范化后的命令行参数分配至位置参数（$1,$2,...)
eval set -- "${ARGS}"

begin_commit_id=""
end_commit_id=""

while [ -n "$1" ]
do
        case "$1" in
            -b|--begin-commit-id)
                begin_commit_id=$2
                shift 2
                ;;
            -e|--end-commit-id)
                end_commit_id=$2
                shift 2
                ;;
            -h|--help)
                show_usage
                shift
                ;;
            --)
                break
                ;;
            *)
                echo "unknown args $1"
                exit -1
                ;;
        esac
done

diff_files=$(git diff ${begin_commit_id} ${end_commit_id} --name-only)

if [[ -z $begin_commit_id || -z $end_commit_id ]]; then
    echo $show_usage
    exit -1
fi
```

