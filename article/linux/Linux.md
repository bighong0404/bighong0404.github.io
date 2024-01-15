# 概述



**一点历史**

Linux是基于Unix系统二次开发, Sun的solari, IBM的AIX也是基于Unix系统实现的, 但后两者只能运行在特定的大型机上, 而Linux可以运行在x86的个人计算机.

<img src="img/image-20231009233720944.png" style="zoom:50%;" />



**优点**

免费, 稳定, 高效,  低成本(软件可裁剪, 内核最小可达几百KB).



**应用领域**

桌面, 服务器, 嵌入式



**内核发布**

https://www.kernel.org/



# 安装



**分区规划**

- boot分区,1G

- swap分区, 与内存大小一致, 用于在内存用满的时候与内存交换, 充当虚拟内存.

- 根分区



# 1. Linux目录结构

- /bin (/usr/bin 、 /usr/local/bin)

  是 Binary 的缩写, 这个目录存放着最经常使用的命令

- /sbin (/usr/sbin 、 /usr/local/sbin)
   s 就是 Super User，表示 存放的是系统管理员使用的系统管理程序。

- /home
   存放普通用户的主目录，在 Linux 中每个用户都有一个自己的目录，

- /root

  该目录为系统管理员，也称作超级权限者的用户主目录

- /lib 

     系统开机所需要最基本的动态链接共享库，其作用类似于 Windows 里的 DLL 文件。几乎所有的应用程序都需要用到这些共享库

- /lost+found 

     默认隐藏.  这个目录一般情况下是空的，当系统非法关机后，这里就存放了一些文件

- /etc (editable text configuration)

     所有的系统管理所需要的配置文件和子目录, 比如安装 mysql 数据库 my.conf
     
- /usr 
   这是一个非常重要的目录，用户的很多应用程序和文件都放在这个目录下，类似与 windows 下的 program files 目录。

- /boot 

   Linux启动时使用的一些核心文件，包括一些连接文件以及镜像文件

- /proc `[不能动] `

   这个目录是一个虚拟的目录，它是系统内存的映射，访问这个目录来获取系统信息

- /srv `[不能动] `

   service 缩写，该目录存放一些服务启动之后需要提取的数据

- /sys `[不能动]`

  这是 linux2.6 内核的一个很大的变化。该目录下安装了 2.6 内核中新出现的一个文件系统sysfs

- /tmp 

   这个目录是用来存放一些临时文件的

- /dev

  类似于 windows 的设备管理器，把所有的硬件设备以文件的形式存储.  `在linux世界, 一切皆为文件`.

- /media

  linux 系统会自动识别一些设备，例如 U 盘、光驱等等，当识别后，linux 会把识别的设备挂载到这个目录下

- /mnt 

   系统提供该目录是为了让用户临时挂载别的文件系统的，我们可以将外部的存储挂载在/mnt/上，然后进入该目录就能看到

- /opt 

   存放要给主机安装额外程序的安装包, 如MSQL安装包。默认为空 

- /usr/local

  存放程序安装包解压后的程序文件, 一般是通过编译源码方式安装的程序

- /var

  这个目录中存放着在不断扩充着的东西，习惯将经常被修改的目录放在这个目录下。包括各种日志文件

- /selinux [security-enhanced linux]
  SELinux 是一种安全子系统,它能控制程序只能访问特定文件, 有三种工作模式，可以自行设置.



# 2. 基础认识



## vim

三种模式: 一般模式, 插入模式, 命令行没事



**快捷方式:**

一般模式下,

- `yy` 复制当前行,   `5yy`从当前行开始向下复制5行
- `p` 粘贴
- `dd`删除当前行, `5dd`从当前行开始向下删除5行
- `G或者shift+g`定位到文档的末行, `gg`定位到文档的首行
- `20G`或者 `20shift+g`, 定位到第20行
- `u`, 撤销上次操作

命令行模式下,

- `/`关键字, 可以搜索关键字, `n`搜索下一个, `N`则反向搜索
- `:set nu` 显示行号, `:set nonu`不现实行号



## 开机、重启

关机&重启命令 7.1.1 基本介绍

- shutdown –h now   立该进行关机
- shudown -h 1          1分钟后关机
- shutdown –r now   现在重新启动计算机
- halt                            关机，作用和上面一样. 
- reboot                      现在重新启动计算机
- sync                         把内存的数据同步到磁盘.



>  不管是重启系统还是关闭系统，首先要运行**sync**命令，把内存中的数据写到磁盘中.
>
> 目前的 shutdown/reboot/halt 等命令均已经在关机前进行了 sync. 



# 3. 用户管理
## 用户

**创建用户** 

- `useradd hyc`,  默认创建用户主目录/home/hyc
- `useradd -d  /home/hyc-test  hyc`, 创建用户hyc并且把用户主目录设置为/home/hyc-test

**设置密码**

- passwd hyc, 给hyc用户设置密码.  **[注意]**用户名必须指定, 否则是给当前用户设置密码  

**删除用户**

- userdel hyc, 删除hyc但保留hyc的主目录
- userdel -r hyc, 删除hyc以及主目录

**查看用户信息**

- id hyc

```shell
[root@centos7 ~]# id root
uid=0(root) gid=0(root) 组=0(root)
[root@centos7 ~]# id milan
id: milan: no such user
```

**切换用户**

- su - hyc, 切换到hyc

从权限高的用户切换到权限低的用户，不需要输入密码，反之需要。
当需要返回到原来用户时，使用exit/logout指令

**查看当前第一次登录用户**

- whoami

- who am i

> 如果su到其他用户, 显示的依旧是最初登录的用户
```shell
[root@centos7 ~]# whoami  
root
[root@centos7 ~]# who am i
root     pts/1        2023-10-17 23:33 (172.16.255.1)
```

  

## 用户组

类似于角色的概念. 

新增组 

- groupadd 组名

删除组

- groupdel 组名

增加用户时直接加上组

- `useradd -g g1 hyc`, 新建用户hyc并且添加到g1组
- `useradd hyc`, 新建用户hyc, 并且添加到hyc组

```shell
[root@centos7 ~]# useradd tom
[root@centos7 ~]# id tom
uid=1001(tom) gid=1001(tom) 组=1001(tom)
```

修改用户组

- `usermod -g g2 hyc`, 把用户hyc挪到g2组
- `usermod -d /home/xxx hyc`, 修改用户的主目录为/home/xxx

**用户和组相关文件**

- /etc/passwd, 用户配置文件

记录用户的各种信息 每行的含义:用户名:口令:用户标识号:组标识号:注释性描述:主目录:是否登录 Shell

```shell
[root@centos7 ~]# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
...
hyc:x:1000:1000:hyc:/home/hyc:/bin/bash
tom:x:1001:1001::/home/tom:/bin/bash
```

- /etc/shadow, 口令的配置文件

每行的含义:登录名:加密口令:最后一次修改时间:最小时间间隔:最大时间间隔:警告时间:不活动时间:失效时间:标志

```shell
[root@centos7 ~]# cat /etc/shadow
root:$6$vMpRahDUrG0t9tfD$UT8d8KLr7XXOimNu.iwNfnXYKNoRIRmnbb1Ak/Lg0UI/8zvnPGzXL1vGzclEYse2BxTndI2RRmRd5Ef02qIHS1::0:99999:7:::
bin:*:17834:0:99999:7:::
daemon:*:17834:0:99999:7:::
adm:*:17834:0:99999:7:::
...
hyc:$6$dVbKmIieo3o9DhL5$yshbiHugsT3Q4U17.rYT8opmIitJCXAeWBUL8XGIEXHYzhMFzQq38og5Lc/c38wdGHaBoJHg8Dt1KlmVC6xlP0::0:99999:7:::
tom:!!:19647:0:99999:7:::
```

- /etc/group, 组配置文件，记录组的信息

每行含义:组名:口令:组标识号:组内用户列表

```shell
[root@centos7 ~]# cat /etc/group
root:x:0:
bin:x:1:
daemon:x:2:
sys:x:3:
...
hyc:x:1000:hyc
tom:x:1001:
```



# 4. 实用指令

## 4.1 指定运行级别

`init 0/1/2/3/4/5/6 `, 设置运行级别

运行级别说明:
 0 :关机
 1 :单用户【找回丢失密码】
 2:多用户状态没有网络服务
 3:多用户状态有网络服务
 4:系统未使用保留给用户
 5:图形界面
 6:系统重启
 **常用运行级别是 3 和 5**, centos7简化为3和5级别, 

`init 3`, 将使安装的图形化失效, 回到命令行模式.

`init 5`, 启用图形化界面

centos7之前, 默认运行级别在文件`/etc/inittab`,

centos7之后, 文件内容变更为说明, 讲解怎么设置系统默认运行级别: 

```shell
[root@centos7 ~]# cat /etc/inittab 
# inittab is no longer used when using systemd.
#
# ADDING CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.
#
# Ctrl-Alt-Delete is handled by /usr/lib/systemd/system/ctrl-alt-del.target
#
# systemd uses 'targets' instead of runlevels. By default, there are two main targets:
#
# multi-user.target: analogous to runlevel 3
# graphical.target: analogous to runlevel 5
#
# To view current default target, run:
# systemctl get-default
#
# To set a default target, run:
# systemctl set-default TARGET.target
#
```

## 4.2 文件类

### tree指令

linux默认不支持tree命令, 需要安装`yum install tree`

```shell
[root@centos7 ~]# tree
bash: tree: 未找到命令...
[root@centos7 ~]# yum install tree   
已加载插件：fastestmirror, langpacks
Determining fastest mirrors
 * base: mirrors.cqu.edu.cn
 * extras: ftp.sjtu.edu.cn
 * updates: ftp.sjtu.edu.cn
.
.
.
已安装:
  tree.x86_64 0:1.6.0-10.el7                                                                                                        
[root@centos7 ~]# tree /mnt/sdb1/
/mnt/sdb1/
├── lost+found
└── my.conf

1 directory, 1 file
```



### touch指令

`touch xxx`, 创建空文件xxx



### cp指令

拷贝文件到指定目录

`cp [选项] source dest`

常用选项: -r, 递归复制



### mv指令

移动文件与目录或重命

`mv oldNameFile newNameFile`



### cat命令

查看文件内容

`cat [选项] 要查看的文件`

常用选项: -n :显示行号

cat 只能浏览文件，而不能修改文件，为了浏览方便，一般会带上 管道命令` | more `

`cat -n /etc/profile | more`


### more命令

于 VI 编辑器的文本过滤器，它以全屏幕的方式按页显示文本文件的内容。more 指令中内置了若 干快捷键(交互的指令)

`more 要查看的文件`

| 快捷键     | 作用                         |
| ---------- | ---------------------------- |
| Enter      | 向下n行，需要定义。默认为1行 |
| Ctrl+F     | 下翻一屏                     |
| 空格键     | 下翻一屏                     |
| Ctrl+B     | 上翻一页                     |
| =          | 输出当前行的行号             |
| ：f        | 输出文件名和当前行的行号     |
| V          | 调用vi编辑器                 |
| !命令      | 调用Shell并执行              |
| G(shift+g) | 到底                         |
| q          | 退出                         |



### less命令

用来分屏查看文件内容，它的功能与 more 指令类似，但是比 more 指令更加强大，支持各种显示终端。less 指令在显示文件内容时，并不是一次将整个文件加载之后才显示，而是根据显示需要加载内容，对于显示大型文件具有较高的效率。

`less 要查看的文件`

| 快捷键     | 作用                                 |
| ---------- | ------------------------------------ |
| [pageup]   | 下翻一屏                             |
| [pagedown] | 上翻一页                             |
| /字符串    | 向下搜索字符串, n向下查找, N向上查找 |
| ?字符串    | 向上搜索字符串, n向上查找, N向下查找 |
| G(shift+g) | 到底                                 |
| q          | 退出                                 |



### head 指令

显示文件的开头部分内容，默认情况下 head 指令显示文件的前 10 行内容

- `head x文件`  输出文件头10行
- `head -n 5 x文件` 输出文件头 5 行内容



### tail 指令

tail 用于输出文件中尾部的内容，默认情况下 tail 指令显示文件的前 10 行内容

- `tail x文件`  输出文件最后10行
- `tail -n 5 x文件` 输出文件最后5 行内容
- `tail -f x文件`实时跟踪文件的更新内容



### echo 指令

传递给 echo 的参数被打印到标准输出中。

`echo` 通常用于 shell 脚本中，用于显示消息或输出其他命令的结果。

- 语法

`echo [选项] [输出内容]`

- 显示变量

  - `echo $PATH`

- 重定向到一个文件

  - `echo 'The only true wisdom is in knowing you know nothing.' >> /tmp/file.txt`

- 把内容管道给其他命令

  - `echo "echo 'hello again' >> /tmp/at-test.txt" | at now +1 minute`, 

    把内容`echo 'hello again' >> /tmp/at-test.txt`管道给命令at



- 如果`-e` 选项，则将解释以下反斜杠转义字符:
  - \ 显示反斜杠字符
  - \a 警报(BEL)
  - \b 显示退格字符
  - \c 禁止任何进一步的输出
  - \e 显示转义字符
  - \f 显示窗体提要字符
  - \n 显示新行
  - \r 显示回车
  - \t 显示水平标签
  - \v 显示垂直标签
  - 这个-E 项禁用转义字符的解释。这是默认值





### \> 指令 和 >> 指令

\> 输出重定向

\>\> 追加

- `ls -l >a.txt`, 列表的内容写入文件 a.txt 中(覆盖写)
- `ls -al >>aa.txt`, 列表的内容追加到文件 aa.txt 的末尾
- `cat 文件1 > 文件2`, 将文件 1 的内容覆盖到文件 2
- `echo "内容">> 文件`, 追加



### ln 指令

软链接也称为符号链接，类似于 windows 里的快捷方式

`ln -s [原文件或目录] [软链接名] `, 给文件创建一个软连接(快捷方式)


`ln -s /root /home/myroot`, 在/home 目录下创建一个软连接 myroot，连接到 /root 目录
`rm /home/myroot`,  删除软连接 myroot



### history 指令

查看已经执行过历史命令,也可以执行历史指令

- `history`, 显示所有的历史命令
- `history 10`, 显示最近使用过的 10 个指令。 
- `!5`, 重新执行编号为5的历史指令



## 4.3 时间日期类

### date指令

**显示当前系统时间**

`date +格式`

```shell
[root@centos7 ~]# date +%Y
2023
[root@centos7 ~]# date +%m
10
[root@centos7 ~]# date +%d
18
[root@centos7 ~]# date '+%Y-%m-%d %H:%M:%S'
2023-10-18 21:04:34
```



**设置系统时间**

`date -s 'yyyyMMdd HH:mm:ss'`



### cal指令

显示日历

- `cal`, 显示当月日历

```shell
[root@centos7 ~]# cal
      十月 2023     
日 一 二 三 四 五 六
 1  2  3  4  5  6  7
 8  9 10 11 12 13 14
15 16 17 18 19 20 21
22 23 24 25 26 27 28
29 30 31
```

- `cal 2023`,显示2023年日历

```shell
[root@centos7 ~]# cal 2023
                               2023                               

        一月                   二月                   三月        
日 一 二 三 四 五 六   日 一 二 三 四 五 六   日 一 二 三 四 五 六
 1  2  3  4  5  6  7             1  2  3  4             1  2  3  4
 8  9 10 11 12 13 14    5  6  7  8  9 10 11    5  6  7  8  9 10 11
15 16 17 18 19 20 21   12 13 14 15 16 17 18   12 13 14 15 16 17 18
22 23 24 25 26 27 28   19 20 21 22 23 24 25   19 20 21 22 23 24 25
29 30 31               26 27 28               26 27 28 29 30 31

        四月                   五月                   六月        
日 一 二 三 四 五 六   日 一 二 三 四 五 六   日 一 二 三 四 五 六
                   1       1  2  3  4  5  6                1  2  3
 2  3  4  5  6  7  8    7  8  9 10 11 12 13    4  5  6  7  8  9 10
 9 10 11 12 13 14 15   14 15 16 17 18 19 20   11 12 13 14 15 16 17
16 17 18 19 20 21 22   21 22 23 24 25 26 27   18 19 20 21 22 23 24
23 24 25 26 27 28 29   28 29 30 31            25 26 27 28 29 30
30
        七月                   八月                   九月        
日 一 二 三 四 五 六   日 一 二 三 四 五 六   日 一 二 三 四 五 六
                   1          1  2  3  4  5                   1  2
 2  3  4  5  6  7  8    6  7  8  9 10 11 12    3  4  5  6  7  8  9
 9 10 11 12 13 14 15   13 14 15 16 17 18 19   10 11 12 13 14 15 16
16 17 18 19 20 21 22   20 21 22 23 24 25 26   17 18 19 20 21 22 23
23 24 25 26 27 28 29   27 28 29 30 31         24 25 26 27 28 29 30
30 31
        十月                  十一月                 十二月       
日 一 二 三 四 五 六   日 一 二 三 四 五 六   日 一 二 三 四 五 六
 1  2  3  4  5  6  7             1  2  3  4                   1  2
 8  9 10 11 12 13 14    5  6  7  8  9 10 11    3  4  5  6  7  8  9
15 16 17 18 19 20 21   12 13 14 15 16 17 18   10 11 12 13 14 15 16
22 23 24 25 26 27 28   19 20 21 22 23 24 25   17 18 19 20 21 22 23
29 30 31               26 27 28 29 30         24 25 26 27 28 29 30
                                              31
```



## 4.4 搜索查找类

### find 指令

从指定目录向下递归地遍历其各个子目录，将满足条件的文件或者目录显示在终端。

`find [搜索范围] [选项]`

| 选项                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| -name pattern          | 按文件名查找，支持使用通配符 `*` 和 `?`                      |
| -type type             | 按文件类型查找，type可以是 `f`（普通文件）、`d`（目录）、`l`（符号链接） |
| -user 用户名           | 按照文件所有者查找                                           |
| -size [+-]size[cwbkMG] | 按文件大小查找，支持使用 `+` 或 `-` 表示大于或小于指定大小，单位可以是 `c`（字节）、`w`（字数）、`b`（块数）、`k`（KB）、`M`（MB）或 `G`（GB） |
| -group groupname       | 按文件所属组查找                                             |
|                        |                                                              |



### locate命令

locate命令用于查找符合条件的文档，他会去保存文档和目录名称的数据库内，查找合乎范本样式条件的文档或目录。

由于 locate 指令基于数据库进行查询，所以第一次运行前，必须使用 `updatedb` 指令创建 locate 数据库, 并且建议定期在闲时执行`updatedb`。

- `locate -i xxx`, 忽略大小写查找包含文件名包含`xxx`的文件
- `locate /etc/xx`,查找/etc目录下的`xx`开头文件



### whereis指令

查找**指令**

```shell
[root@centos7 ~]# whereis tar
tar: /usr/bin/tar /usr/include/tar.h /usr/share/man/man5/tar.5.gz /usr/share/man/man1/tar.1.gz
```



### 管道符号 |

管道符“|”表示将前一个命令的处理结果输出传递给后面的命令处理。

经常跟文本处理三剑客一起使用

`ps -ef|grep xxxx`



### 文本处理三剑客

awk、grep、sed是linux操作文本的三大利器，合称文本三剑客。三者的功能都是处理文本，但侧重点各不相同，

- grep适合单纯的查找或匹配文本
- sed适合编辑匹配到的文本
- awk适合对格式化文本进行处理分析, 并生成报告。

其中属awk功能最强大，但也最复杂。



#### grep指令

grep全称是 `Global Regular Expression Print`, 使用正则表达式搜索文本，并把匹配的行打印出来。

**语法**

`grep [option] pattern file`,  PATTERN支持正则

**option说明**

- -A<行数 x>：除了显示符合范本样式的那一列之外，并显示该行之后的 x 行内容。
- -B<行数 x>：除了显示符合样式的那一行之外，并显示该行之前的 x 行内容。
- -C<行数 x>：除了显示符合样式的那一行之外，并显示该行之前后的 x 行内容。
- -c：统计匹配的行数
- -e ：实现多个选项间的逻辑or 关系
- -F: 相当于`fgrep`命令，就是将`pattern`视为固定字符串, 而不是正则表达式
- -i:  --ignore-case, 忽略字符大小写的差别。
- -n：显示匹配的行号
- -o：仅显示匹配到的字符串
- -q： 静默模式，不输出任何信息. 有匹配的内容则立即返回状态值 0。一般用在脚本中的 `if` 判断里面
- -s：不显示错误信息。
- -v：显示不被 `pattern` 匹配到的行，相当于[^] 反向匹配
- -w ：匹配 整个单词



#### sed指令

`sed` 命令全称是`Stram editor流编辑器`, 作用是利用脚本来处理文本文件。

**语法**

`sed [-hnV][-e <script>][-f <script文件>][文本文件]`

**参数说明**

- `-e <script>`: `--expression=<script>` , 用指定的 `script脚本` 来处理输入的文本文件，
  - 这个`-e`可以省略，直接写`script脚本`。
  - `script脚本`经常用引号括起来, `sed '3a新内容' xxxfile`
- `-f <script文件>`: `--file=<script文件>`, 用指定的 `script脚本文件` 来处理输入的文本文件。
- `-h`: `--help`显示帮助。
- `-n` :  `--quiet` 或 `--silent` 仅显示被 `script`修改的行。
- `-V` :  `--version` 显示版本信息。
- `-i[后缀]`:  替换和备份源文件, 假设`sed`处理文件`test.txt`,  
  - `sed -i -e '/MacOS/c\Windows' test.txt` , 把sed结果写到`test.txt`中
  - `sed -ibackup -e '/MacOS/c\Windows' test.txt`, 把文件备份为`test.txtbackup,`把sed结果写到`test.txt`中


**script脚本动作**

- a：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)
  - 脚本动作和参数可以使用`\`来隔开, 方便阅读,   `sed "3a\新内容" xxxfile`
  - 脚本动作前面是数字, 则是匹配指定第n行, `sed '1,4a\newline' file`在第1~4行后面新增一行newLine
  - 脚本动作前面是`/text/`, 则是匹配`text`字符串的行
- c：整行取代， `sed '/Linux/c\MacOS' sedTest.txt`, 把匹配到Linux的**行**替换为MacOS
- d：删除，把匹配到的行删除
- i：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
- p：打印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
- s：字符串代替，可以搭配正规表示法, 
- `sed "1,20s/a/A/g" file`, 在第1~12行, 把每一行的a替换为A
  - ```sed "s/a/A/3g" file```, 在每一行中, 第三个匹配到的a替换为A





**a新增内容示例**

`a`新增,  `sed 3anewLine [file]`, 在第三行下面, 新增(`a`)一行内容为`newLine`的文本. 

注意，这个只是将文字处理了，没有写入到文件里，文件里还是之前的内容。

```shell
[root@centos7 ~]# cat sedTest.txt 
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 ~]# sed 3a\newLine sedTest.txt                           
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  # 在第三行后面新增一行newLine
newLine
Linux test
```

`sed -e "/Linux/a\newline" sedTest.txt`,  匹配字符串Linux

```shell
[root@centos7 ~]# cat sedTest.txt                      
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 ~]# sed -e "/Linux/a\newline" sedTest.txt
HELLO LINUX!  
Linux is a free unix-type opterating system. #匹配到Linux, 往下新增一行newline
newline
This is a linux testfile!  
Linux test #匹配到Linux, 往下新增一行newline
newline
```



**i插入内容示例**

```shell
[root@centos7 ~]# cat sedTest.txt                      
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 ~]# sed 3i\newline sedTest.txt 
HELLO LINUX!  
Linux is a free unix-type opterating system.  
newline
This is a linux testfile!  #原本是第三行, 在此行前面插入一行newline
Linux test
```



**d删除示例**

```shell
[root@centos7 ~]# cat sedTest.txt 
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 ~]# sed '/Linux/d' sedTest.txt  #把匹配到`Linux`的行删除
HELLO LINUX!  
This is a linux testfile! 
[root@centos7 ~]# sed '3d' sedTest.txt  # 把第3行删除
HELLO LINUX!  
Linux is a free unix-type opterating system.  
Linux test
```



**c替换示例**

```shell
[root@centos7 root_desktop]# cat sedTest.txt 
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 root_desktop]# sed '/Linux/c\MacOS' sedTest.txt 
HELLO LINUX!  
MacOS  # 把匹配到Linux的行内容都替换为MacOS
This is a linux testfile!  
MacOS
```



**s替换示例**

```shell
[root@centos7 root_desktop]# cat sedTest.txt 
HELLO LINUX!  
Linux is a free unix-type opterating system.  
This is a linux testfile!  
Linux test
[root@centos7 root_desktop]# sed "s/Linux/MacOS/g" sedTest.txt 
HELLO LINUX!  
MacOS is a free unix-type opterating system.   # 仅仅把Linux替换为MacOS
This is a linux testfile!  
MacOS test
```



**多个匹配**

依次执行两次script匹配

```shell
sed -e 's/Linux/Windows/g' -e 's/Windows/Mac OS/g' sedTest.txt 
```



**sed执行完后写入文件**

上面介绍的所有文件操作都支持在缓存中处理然后打印到控制台，实际上没有对文件修改。想要保存到原文件的话可以用`> file`或者`-i`来保存到文件



#### awk指令

`awk`取自于它的创始人`Alfred Aho 、Peter Weinberger 和 Brian Kernighan` 姓氏的首个字母。 实际上 AWK 的确拥有自己的语言： AWK 程序设计语言 ， 三位创建者已将它正式定义为“**样式扫描和处理语言**”。

awk是一个强大的文本**分析**工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。

**语法**

`awk [选项参数] 'script' var=value file(s)`
`awk [选项参数] -f scriptfile var=value file(s)`

**参数说明**：

- `-F 分隔符`:  `--field-separator 分隔符` 指定输入文件折分隔符(默认空格或tab)，分隔符可以是一个字符串或者正则表达式
- `-v var=value`: `--asign var=value` 赋值一个用户定义变量。
- `-f scripfile`: `--file scriptfile` 从脚本文件中读取awk命令。



**示例**

```shell
[root@centos7 ~]# cat awkDemo.txt 
2 this is a test
3 Are you like awk
This's a test
10 There are orange,apple,mongo
[root@centos7 ~]# awk '{print $1,$4}' awkDemo.txt    # $1表示第一个单次, $0代表整行
2 a
3 like
This's 
10 orange,apple,mongo
[root@centos7 root_desktop]# awk -F '[ ,]'  '{print $1,$2,$5}' awkDemo.txt   #以空格和,作为分隔符
2 this test
3 Are awk
This's a 
10 There apple
[root@centos7 root_desktop]# awk '/^This/' awkDemo.txt  # 匹配This
This's a test
```



**内置变量**

- `FILENAME`：当前文件名
- `NF`：分割后的字段数量
- `FS`：字段分隔符，默认是空格和制表符。
- `RS`：行分隔符，用于分割每一行，默认是换行符。
- `OFS`：输出字段的分隔符，用于打印时分隔字段，默认为空格。
- `ORS`：输出记录的分隔符，用于打印时分隔记录，默认为换行符。
- `OFMT`：数字输出的格式，默认为`％.6g`。

```shell
[root@centos7 ~]# cat awkDemo.txt 
2 this is a test
3 Are you like awk
This's a test
10 There are orange,apple,mongo
[root@centos7 ~]# awk '{print NF}' awkDemo.txt  # 切分后的段数
5
5
3
4
[root@centos7 ~]# awk '{print $NF}' awkDemo.txt  # $段数, 相当于取最后一段
test
awk
test
orange,apple,mongo
```



**内置函数**

内置函数，方便对原始数据的处理

- `toupper()`：字符转为大写。
- `tolower()`：字符转为小写。
- `length()`：返回字符串长度。
- `substr()`：返回子字符串。
- `sin()`：正弦。
- `cos()`：余弦。
- `sqrt()`：平方根。
- `rand()`：随机数。

```shell
[root@centos7 ~]# cat awkDemo.txt 
2 this is a test
3 Are you like awk
This's a test
10 There are orange,apple,mongo
[root@centos7 ~]# awk '{print $NF}' awkDemo.txt  # $段数, 相当于取最后一段
test
awk
test
orange,apple,mongo
[root@centos7 ~]# awk '{print toupper($NF)}' awkDemo.txt                          
TEST
AWK
TEST
ORANGE,APPLE,MONGO
```



**if 语句**

`awk`提供了`if`结构，用于编写复杂的条件

```shell
[root@centos7 ~]# cat awkDemo.txt 
2 this is a test
3 Are you like awk
This's a test
10 There are orange,apple,mongo
[root@centos7 ~]# awk '{print $1,$2}' awkDemo.txt    
2 this
3 Are
This's a
10 There
[root@centos7 ~]# awk '{if ($2 > "t") print $1}' awkDemo.txt 
2
```

把每一行按照空格分割之后，如果第二个单词大于`t`，就输出第一个单词。这里对字符的大小判断应该是基于字符长度和 unicode 编码。



### 其他常用的文本处理命令

#### cut

cut命令主要用来切割字符串，可以对输入的数据进行切割然后输出。

**语法**

`cut [参数] [文件]`

**参数说明**

- -b: 以字节为单位进行分割
- -c: 以字符为单位进行分割 
- -d: 自定义分隔符，默认为制表符 
- -f: 显示cut后的第几段字符

```shell
[root@centos7 ~]# cat t.log 
http://192.168.200.10/index1.html 
http://192.168.200.10/index2.html
http://192.168.200.20/index1.html
http://192.168.200.30/index1.html
http://192.168.200.40/index1.html
http://192.168.200.30/order.html
http://192.168.200.10/order.html
[root@centos7 ~]# cat t.log | cut -d '/' -f 3      
192.168.200.10
192.168.200.10
192.168.200.20
192.168.200.30
192.168.200.40
192.168.200.30
192.168.200.10
```



### sort

sort 对指定文本排序, 默认将文本文件的第一列以 ASCII 码的次序排列，并将结果输出到标准输出。

**格式**

`sort [参数][文件][-k field1[,field2]]`

**参数**

- `-b`: 忽略每行前面开始出的空格字符。
- `-c`:  检查文件是否已经按照顺序排序。
- `-d`:  排序时，处理英文字母、数字及空格字符外，忽略其他的字符。
- `-f`:  排序时，将小写字母视为大写字母。
- `-i`:  排序时，除了040至176之间的ASCII字符外，忽略其他的字符。
- `-m`:  将几个排序好的文件进行合并。
- `-M`:  将前面3个字母依照月份的缩写进行排序。
- `-n`:  依照数值的大小排序。
- `-u`:  意味着是唯一的(unique)，输出的结果是去完重了的。
- `-o<输出文件> `: 将排序后的结果存入指定的文件。
- `-r`:  以相反的顺序来排序。
- `-t<分隔字符>`:  指定排序时所用的栏位分隔字符。
- `-k [field]`: `sort [file] -k [field1]`, 按照指定列排序



#### uniq

用于检查及删除文本文件中重复出现的行列，一般与 sort 命令结合使用。

**格式**

`uniq [参数][输入文件][输出文件]`

**参数**

- `-c`: 在每列旁边显示该行重复出现的次数。
- `-d`:  仅显示重复出现的行列。
- `-f<栏位>`:  忽略比较指定的栏位。
- `-s<字符位置>`:  忽略比较指定的字符。
- `-u`: 仅显示出一次的行列。
- `-w<字符位置>`: 指定要比较的字符。
- `[输入文件] `: 指定已排序好的文本文件。不指定则从标准读取数据。
- `[输出文件]`: 指定输出的文件。不指定则将内容显示到标准输出设备（显示终端）。

```shell
[root@centos7 practise]# cat t.log
http://192.168.200.10/index1.html 
http://192.168.200.10/index2.html
http://192.168.200.20/index1.html
http://192.168.200.30/index1.html
http://192.168.200.40/index1.html
http://192.168.200.30/order.html
http://192.168.200.10/order.html
[root@centos7 practise]# cat t.log | cut -d '/' -f 3| sort | uniq -c | sort -nr
      3 192.168.200.10
      2 192.168.200.30
      1 192.168.200.40
      1 192.168.200.20
```



#### diff

逐行比较文件

```shell
[root@centos7 practise]# diff t.log t2.log -y -W 100 # 并列显示比较结果, 并以100的列宽展示
http://192.168.200.10/index1.html             | http://192.168.200.11/index1.html 
http://192.168.200.10/index2.html             | http://192.168.200.22/index1.html
http://192.168.200.20/index1.html             | http://192.168.200.30/index2.html
http://192.168.200.30/index1.html             | http://192.168.200.42/index1.html
http://192.168.200.40/index1.html             | diff line
http://192.168.200.30/order.html              | http://192.168.200.33/order.html
                                              > http://192.168.200.10/order.html
http://192.168.200.10/order.html                http://192.168.200.10/order.html
```





paste：合并文件文本行。
join：基于某个共享字段来联合两个文件的文本行。
comm：逐行比较两个已经排好序的文件。

patch：对原文件打补丁。
tr：转换或删除字符。

aspel：交互式拼写检查器。.





## 4.5 压缩和解压

### gzip/gunzip 指令

gzip压缩文件, *不压缩目录*, gunzip解压

- `gzip xxx.txt`, 压缩文件xxx.txt为xxx.txt.gz的压缩文件

- `gzip -r /home/test`, 递归压缩/home/test目录下的每一个文件 

  

### zip/unzip 指令

zip 用于压缩文件/文件夹为一个压缩包， unzip 用于解压的

`zip [选项] 文件/目录`

- `zip X.zip x文件`,压缩x文件为x.zip 
- `zip -r X.zip 目录`, 压缩目录为x.zip
- 选项-v, 显示详细的压缩过程信息
- `zip -dv x.zip 文件1` ,  从压缩文件内删除文件1

`unzip [选项] x.zip`

- `unzip -d X.zip`, 把x.zip解压到指定目录

- 选项-l, 显示压缩文件内所包含的文件。
- 选项 -v,  查看压缩文件目录信息，但是不解压该文件



### tar指令

打包指令，最后打包后的文件是 `.tar.gz` 的文件

`tar [选项] XXX.tar.gz 打包的内容1 打包内容2`

| 选项 | 说明                  |
| ---- | --------------------- |
| -c   | 建立新的备份文件      |
| -C   | 切换目录, 解压时候用  |
| -v   | verbose, 显示详细信息 |
| -f   | 执行打包后的文件名    |
| -z   | 打包的同时压缩        |
| -x   | 解压.tar文件          |

- `tar -czvf test.tar.gz /home/test /home/test1` , 把`/home/test`, `/home/test1`目录压缩打包成test.tar.gz文件到当前目录
- `tar -xzvf test.tar.gz `,解压test.tar.gz文件到当前目录
- `tar -xzvf test.tar.gz -C /home` ,解压test.tar.gz文件到/home目录下



# 5. 组管理和权限管理

## 文件权限

```shell
[root@centos7 folder]# ll
总用量 12
-rw-r--r--. 1 root root    0 10月 19 23:23 1.txt
-rw-r--r--. 1 root root   51 10月 19 22:06 1.txt.gz
-rwxr-xr-x. 1 root root    5 10月 19 23:25 cmd.sh     <---以此文件为例
drwxr-xr-x. 2 root root 4096 10月 19 23:22 tmp
```

| 第0未                                                        | 123位                                                        | 456位                   | 789位                   | 1                                                      | root   | root             | 数值 | 时间          | 文件名 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------- | ----------------------- | ------------------------------------------------------ | ------ | ---------------- | ---- | ------------- | ------ |
| -<br />类型                                                  | rwx<br />所有者权限                                          | r-x<br />同一组用户权限 | r-x<br />其他组用户权限 | 数值                                                   | 所有者 | 所在组           | 大小 | 10月 19 23:23 | cmd.sh |
| - 表示普通文件 <br />d 表示目录 <br />l 表示符号链接 <br />c 表示字符设备文件,鼠标键盘 <br />b 表示块设备文件, 磁盘等 <br />s 表示套接字文件 <br />p 表示管道文件 | r 表示可读权限 <br />w 表示修改权限<br />x 表示执行文件/进入目录权限 <br />\- 表示没有对应权限 | 同左                    | 同左                    | 目录: 表示子目录个数, <br />文件: 指向次文件的链接个数 | 所有者 | 默认是创建者的组 | 字节 | 最后修改时间  |        |



## 修改权限chmod



第一种方式: + 、-、= 变更权限, 

*u:所有者 g:所有组 o:其他人 a:所有人(u、g、o 的总和)*

- `chmod u=rwx,g=rx,o=x 文件/目录`, 给u(所有者), g(组), o(其他用户)设置对应权限
- `chmod o+w 文件/目录名` ,  给o(其他人)添加w(可读)权限
- `chmod +w 文件/目录名` ,  给所有人添加x(执行)权限
- `chmod a-x 文件/目录名`, 给a(所有人)移出x(执行)权限



第二种方式:通过数字变更权限

r=4 w=2 x=1 

rwx=4+2+1=7

`chmod 751 文件1` 等效于 `chmod u=rwx,g=rx,o=x 文件1`



### 修改所有者chown

- `chown tom a.txt`  修改a.txt的所有者为tom
- `chown tom:g1 a.txt` 改变a.txt的所有者和所在组
- `chown -R tom /home/test` 修改/home/test以及下级所有文件和目录所有者为tom



### 修改文件/目录所在组chgrp

- `chgrp g1 a.txt`, 修改文件组为g1
- `chgrp -R g1 /home/test`, 把/home/test以及下级所有文件和目录的所在组都修改成g1





# 6. 定时任务调度

## crond/crontab

**crond**是Linux系统用来定期执行命令或指定程序的服务的一种服务或软件。crond服务会定期（默认一分钟检查一次）检查系统中是否有要执行的任务工作。

**crontab**是用于设置周期性被执行的指令，该命令从标准输入设备读取指令，并将其存放于“crontab”文件中，以供之后读取与执行。  



**crontab语法**

- ` crontab –e` 当前用户编辑调度任务
  - */5 * * * * `command`, 每天5分钟点执行命令
- `crontab -l` 查看所有当前用户的任务
- `crontab -r` 删除所有当前用户的任务, 若仅要移除一项，请用` -e` 去编辑
- 系统的配置文件: `/etc/crontab`
  - `crontab -e` 命令是针对用户的cron设计的，`crontab -e` 其实是`/usr/bin/crontab`这个文件
  - 系统的例行性任务使用配置文件`/etc/crontab`

```shell
[root@centos7 ~]# cat /etc/crontab   
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed

1 * * * * root run-parts /etc/cron.hourly    	//每小时1分执行/etc/cron.hourly目录下的命令
0 1 * * * root run-parts /etc/cron.daily			//每天1点执行/etc/cron.daily目录下的命令
0 4 * * 0 root run-parts /etc/cron.weekly			//每周日4点执行/etc/cron.weekly目录下的命令
0 4 1 * * root run-parts /etc/cron.monthly		//每月1日4点执行/etc/cron.monthly目录下的命令
```



crontab时间格式, 5段符号

| 代表意义 | 分钟 | 小时 | 日期 | 月份 | 周             |
| -------- | ---- | ---- | ---- | ---- | -------------- |
| 数字范围 | 0-59 | 0-23 | 1-31 | 1-12 | 0-6(0/7是周日) |

| 特殊字符 | 代表意义                                                     |
| -------- | ------------------------------------------------------------ |
| *(星号)  | 代表任何时刻都接受的意思！                                   |
| ,(逗号)  | 代表分隔时段的意思。<br />0 3,6 * * * command, 每第3和6分钟执行command |
| -(减号)  | 代表一段时间范围内，<br />20 8-12 * * * command,  8 点到 12 点之间的第20分 |
| /n(斜线) | 每隔 n 单位间隔，<br />*/5 * * * * command, 每五分钟进行一次 |

```shell
systemctl reload crond.service #重启crontab
systemctl start crond.service 
systemctl stop crond.service
systemctl restart crond.service

# 如果不支持systemctl命令
service crond start    //启动服务
service crond stop     //关闭服务
service crond restart  //重启服务
service crond reload   //重新载入配置
service crond status   //查看服务状态 
```



## at

at 命令是一次性定时计划任务，执行完一个任务后不再执行此任务了.

at 的守护进程 atd 会以后台模式运行，每60秒检查作业队列来运行。

在使用at命令的时候，一定要保证atd进程的启动,使用`ps -ef | grep atd` 查看` atd` 是否在运行

- 语法

  ```css
  at [选项] [日期时间]
  ```
  - 选项
  
  ```diff
  -f：指定包含具体指令的任务文件
  -q：指定新任务的队列名称
  -l：显示待执行任务的列表
  -d：删除指定的待执行任务
  -m：任务执行完成后向用户发送 E-mail
  ```
  
  - 日期时间
  
  ```undefined
  YYMMDDhhmm[.ss]（缩写年、月、日、小时、分钟[秒]）
  CCYYMMDDhhmm[.ss]（完整年、月、日、小时、分钟和[秒]）
  now
  midnight
  noon
  teatime（下午4点）
  AM
  PMa
  
  也可以添加一个加号 ( + ) 使它们相对于现在：
  minutes
  hours
  days
  weeks
  months
  years
  ```



推荐使用管道命令添加at命令

`echo "echo 'hello again' >> /tmp/at-test.txt" | at now +1 minute`



# 7. 磁盘分区与挂载

## lsblk命令

lsblk的全称是“list block”，列出所有可用块设备的信息，而且还能显示他们之间的依赖关系，但是它不会列出RAM盘的信息。

块设备有硬盘，闪存盘，CD-ROM等等。

```shell
[root@centos7 ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   15G  0 disk 
├─sda1   8:1    0    1G  0 part /boot
├─sda2   8:2    0    2G  0 part [SWAP]
└─sda3   8:3    0   12G  0 part /
sr0     11:0    1 1024M  0 rom  
```



### 硬盘说明

Linux 硬盘分 IDE 硬盘和 SCSI 硬盘，目前基本上是 **SCSI** 硬盘.

- 对于SCSI硬盘则标识为“sdx~”，SCSI硬盘是用“sd”来表示分区所在设备的类型的. “x”为盘号(a为基本盘，b 为基本从属盘，c 为辅助主盘，d 为辅助从属盘),“~”代表分区，前四个分区用数字 1 到 4 表示，它们是主分区或扩展分区，从 5 开始就是逻辑分区.

  以SCSI硬盘为例, `sda2`表示第一块SCSI盘的第二个主分区或拓展分区. `sdb2`表示为第二个 IDE 硬盘上的第二个主分区或扩展分区

- 对于IDE硬盘，驱动器标识符为“hdx~”,其中“hd”表明分区所在设备的类型，其他规则与SCSI硬盘一致。

> IDE和SCSI的区别: 
>
> - IDE的工作方式需要CPU的全程参与，CPU读写数据的时候不能再进行其他操作；而SCSI接口，则完全通过独立的高速的SCSI卡来控制数据的读写操作，CPU就不必浪费时间进行等待，可以提高系统的整体性能。
> - SCSI的扩充性比IDE大，一般每个IDE系统可有2个IDE通道，总共连4个IDE设备，而SCSI接口可连接7—15个设备，比IDE要多很多。 连接的线缆也远长于IDE。
> - 性价比上，在当时IDE更具有优势，SCSI接口价格一般会比IDE接口贵一些。



## 挂载一个新磁盘

步骤: 

1. 虚拟机添加硬盘, 添加完后重启linux, lsblk命令查看新磁盘, 如下`sdb`就是新磁盘

   ```shell
   [root@centos7 ~]# lsblk
   NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   sda      8:0    0   15G  0 disk 
   ├─sda1   8:1    0    1G  0 part /boot
   ├─sda2   8:2    0    2G  0 part [SWAP]
   └─sda3   8:3    0   12G  0 part /
   sdb      8:16   0    1G  0 disk 
   sr0     11:0    1 1024M  0 rom
   [root@centos7 ~]# ls /dev/ |grep sdb   // 设备都在/dev目录下, 因此新分区是/dev/sdb
   sdb
   ```
   `fdisk -l`也可以查看磁盘分区表
   
   ```shell
   [root@centos7 ~]# fdisk -l
   
   磁盘 /dev/sda：16.1 GB, 16106127360 字节，31457280 个扇区
   Units = 扇区 of 1 * 512 = 512 bytes
   扇区大小(逻辑/物理)：512 字节 / 512 字节
   I/O 大小(最小/最佳)：512 字节 / 512 字节
   磁盘标签类型：dos
   磁盘标识符：0x000b6a64
   
      设备 Boot      Start         End      Blocks   Id  System
   /dev/sda1   *        2048     2099199     1048576   83  Linux
   /dev/sda2         2099200     6293503     2097152   82  Linux swap / Solaris
   /dev/sda3         6293504    31451135    12578816   83  Linux
   
   磁盘 /dev/sdb：1073 MB, 1073741824 字节，2097152 个扇区
   Units = 扇区 of 1 * 512 = 512 bytes
   扇区大小(逻辑/物理)：512 字节 / 512 字节
   I/O 大小(最小/最佳)：512 字节 / 512 字节
   磁盘标签类型：dos
   磁盘标识符：0xd7099d78
   ```
   
   


2. 分区

    使用分区命令 `fdisk /dev/sdb`, 按提示操作, 完成后记得输入w命令写入磁盘保存设置.

    ```shell
    [root@centos7 ~]# fdisk /dev/sdb
    欢迎使用 fdisk (util-linux 2.23.2)。
    
    更改将停留在内存中，直到您决定将更改写入磁盘。
    使用写入命令前请三思。
    
    Device does not contain a recognized partition table
    使用磁盘标识符 0xd7099d78 创建新的 DOS 磁盘标签。
    
    命令(输入 m 获取帮助)：m
    命令操作
       a   toggle a bootable flag
       b   edit bsd disklabel
       c   toggle the dos compatibility flag
       d   delete a partition
       g   create a new empty GPT partition table
       G   create an IRIX (SGI) partition table
       l   list known partition types
       m   print this menu
       n   add a new partition
       o   create a new empty DOS partition table
       p   print the partition table
       q   quit without saving changes
       s   create a new empty Sun disklabel
       t   change a partition's system id
       u   change display/entry units
       v   verify the partition table
       w   write table to disk and exit
       x   extra functionality (experts only)
    
    命令(输入 m 获取帮助)：n
    Partition type:
       p   primary (0 primary, 0 extended, 4 free)
       e   extended
    Select (default p): p 		// 主分区
    分区号 (1-4，默认 1)：1 			//1个分区	
    起始 扇区 (2048-2097151，默认为 2048)：
    将使用默认值 2048
    Last 扇区, +扇区 or +size{K,M,G} (2048-2097151，默认为 2097151)：
    将使用默认值 2097151
    分区 1 已设置为 Linux 类型，大小设为 1023 MiB
    
    命令(输入 m 获取帮助)：w   	//w   write table to disk and exit, 写入磁盘保存设置, 并退出
    The partition table has been altered!
    
    Calling ioctl() to re-read partition table.
    正在同步磁盘。	
    ```

3. 格式化分区

    命令`mkfs -t ext4 /dev/sdb1`

    ```shell
    [root@centos7 ~]# mkfs -h
    用法：
     mkfs [选项] [-t <类型>] [文件系统选项] <设备> [<大小>]
    
    选项：
     -t, --type=<类型>  文件系统类型；若不指定，将使用 ext2
         fs-options     实际文件系统构建程序的参数
         <设备>         要使用设备的路径
         <大小>         要使用设备上的块数
     -V, --verbose      解释正在进行的操作；
                          多次指定 -V 将导致空运行(dry-run)
     -V, --version      显示版本信息并退出
                          将 -V 作为 --version 选项时必须是惟一选项
     -h, --help         显示此帮助并退出
    
    更多信息请参阅 mkfs(8)。
    [root@centos7 ~]# mkfs -t ext4 /dev/sdb1
    mke2fs 1.42.9 (28-Dec-2013)
    文件系统标签=
    OS type: Linux
    块大小=4096 (log=2)
    分块大小=4096 (log=2)
    Stride=0 blocks, Stripe width=0 blocks
    65536 inodes, 261888 blocks
    13094 blocks (5.00%) reserved for the super user
    第一个数据块=0
    Maximum filesystem blocks=268435456
    8 block groups
    32768 blocks per group, 32768 fragments per group
    8192 inodes per group
    Superblock backups stored on blocks: 
            32768, 98304, 163840, 229376
    
    Allocating group tables: 完成                            
    正在写入inode表: 完成                            
    Creating journal (4096 blocks): 完成
    Writing superblocks and filesystem accounting information: 完成
    ```

4. 挂载

    `mount`命令把分区`/dev/sdb1`挂载到`/mnt/sdb1`目录

    ```shell
    [root@centos7 ~]# cd /mnt/
    [root@centos7 mnt]# mkdir sdb1
    [root@centos7 mnt]# mount /dev/sdb1 /mnt/sdb1
    [root@centos7 sdb1]# lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda      8:0    0   15G  0 disk 
    ├─sda1   8:1    0    1G  0 part /boot
    ├─sda2   8:2    0    2G  0 part [SWAP]
    └─sda3   8:3    0   12G  0 part /
    sdb      8:16   0    1G  0 disk 
    └─sdb1   8:17   0 1023M  0 part /mnt/sdb1    //显示已经挂载到/mnt/sdb1
    sr0     11:0    1 1024M  0 rom
    ```

    如果想更换挂载的目录, 需要先卸载(`unmount`命令), 再重新挂载`mount`

5. 设置永久挂载

    将分区挂载写入`/etc/fstab`文件，防止主机重启后分区丢失的问题. 添加完成后执行 `mount –a` 即刻生效

    ```shell
    [root@centos7 ~]# cat /etc/fstab 
    
    #
    # /etc/fstab
    # Created by anaconda on Tue Oct 10 01:05:33 2023
    #
    # Accessible filesystems, by reference, are maintained under '/dev/disk'
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
    #
    UUID=919b081c-a940-474b-a042-6e298514a6c8 /                       ext4    defaults        1 1
    UUID=b9cc762e-1537-40df-93a6-aa2da3376ffc /boot                   ext4    defaults        1 2
    UUID=25ce7747-ff98-49f0-acf1-7220ebf9b5bc swap                    swap    defaults        0 0
    /dev/sdb1      /mnt/sdb1    ext4  defaults        0 0     // 这一行
    ```

    添加配置`/dev/sdb1      /mnt/sdb1    ext4  defaults        0 0`, 使用分区uuid也可以, 分区uuid通过命令`lsblk -f`查看



# 8. 进程管理

## ps命令

ps 命令是用来查看目前系统中进程执行情况.

`语法`:

​	ps [option]

`常用option`

- -a,显示当前终端的所有进程

- -u, 以用户形式显示进程信息

- -x,显示进程参数

- -e 显示所有进程。

- -f 全格式输出

  

**`ps -aux`输出内容说明, BSD格式输出**

```shell
[root@centos7 ~]# ps -aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.2 128276  5172 ?        Ss   11月19   0:24 /usr/lib/systemd/systemd --switched-root --system --deserialize 
root          2  0.0  0.0      0     0 ?        S    11月19   0:00 [kthreadd]
root          3  0.0  0.0      0     0 ?        S    11月19   0:04 [ksoftirqd/0]
root       7598  0.0  0.0  53856   376 ?        S    11月19   0:00 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.c
root       7684  0.0  0.1  91628  2160 ?        Ss   11月19   0:01 /usr/libexec/postfix/master -w
postfix    7696  0.0  0.1  91800  3832 ?        S    11月19   0:00 qmgr -l -t unix -u
root       7861  0.0  0.3 430364  6300 ?        Ssl  11月19   0:01 /usr/libexec/upowerd
root       7928  0.0  0.1 398564  3632 ?        Ssl  11月19   0:00 /usr/libexec/boltd
root       7932  0.0  0.4 562372  9560 ?        Ssl  11月19   0:06 /usr/libexec/packagekitd
root       7942  0.0  0.1  78560  2740 ?        Ss   11月19   0:00 /usr/sbin/wpa_supplicant -u -f /var/log/wpa_supplicant.log -c /e
colord     8026  0.0  0.2 419712  5208 ?        Ssl  11月19   0:00 /usr/libexec/colord
postfix   62255  0.0  0.2  91732  4088 ?        S    21:01   0:00 pickup -l -t unix -u
root      62732  0.0  0.0      0     0 ?        S    21:50   0:00 [kworker/u256:2]
root      63088  0.0  0.0      0     0 ?        S    22:20   0:00 [kworker/u256:1]
root      63110  0.0  0.0      0     0 ?        S    22:22   0:00 [kworker/0:1]
root      63121  0.0  0.0 110484  1000 pts/0    S+   22:23   0:00 more
root      63123  0.0  0.2 160848  5644 ?        Ss   22:23   0:00 sshd: root@pts/1
root      63128  0.0  0.1 116652  3288 pts/1    Ss   22:23   0:00 -bash
root      63175  0.0  0.0 110484  1004 pts/1    S+   22:23   0:00 more
root      63176  0.0  0.2 160848  5648 ?        Ss   22:23   0:00 sshd: root@pts/2
root      63181  0.0  0.1 116652  3304 pts/2    Ss   22:23   0:00 -bash
root      63281  0.0  0.0      0     0 ?        S    22:27   0:00 [kworker/0:0]
root      63328  0.0  0.0 107952   616 ?        S    22:30   0:00 sleep 60
root      63330  0.0  0.0 157444  1928 pts/2    R+   22:31   0:00 ps -aux
```

- USER: 行程拥有者
- PID: pid
- %CPU: 占用的 CPU 使用率
- %MEM: 占用的内存使用率
- VSZ: 占用的虚拟内存大小, KB
- RSS: 占用的物理内存大小, KB
- TTY: 终端的次要装置号码 (minor device number of tty)
- STAT: 该行程的状态:
  - D: 无法中断的休眠状态 (通常 IO 的进程)
  - R: 正在执行中
  - S: 静止状态
  - T: 暂停执行
  - Z: 不存在但暂时无法消除
  - W: 没有足够的记忆体分页可分配
  - <: 高优先序的行程
  - N: 低优先序的行程
  - L: 有内存分页分配并锁在内存 (实时系统或捱A I/O)
- START: 行程开始时间
- TIME: 执行的时间
- COMMAND:所执行的指令



**`ps -ef`输出内容说明**, 标准格式

```shell
[root@centos7 ~]# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  0 11月19 ?      00:00:24 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
root          2      0  0 11月19 ?      00:00:00 [kthreadd]
root          3      2  0 11月19 ?      00:00:04 [ksoftirqd/0]
root          5      2  0 11月19 ?      00:00:00 [kworker/0:0H]
root          7      2  0 11月19 ?      00:00:00 [migration/0]
root          8      2  0 11月19 ?      00:00:00 [rcu_bh]
root          9      2  0 11月19 ?      00:00:16 [rcu_sched]
root         10      2  0 11月19 ?      00:00:00 [lru-add-drain]
root         11      2  0 11月19 ?      00:00:19 [watchdog/0]
hyc       34795  34157  0 11月19 ?      00:00:00 abrt-applet
hyc       34818  34763  0 11月19 ?      00:00:00 /usr/libexec/evolution-addressbook-factory-subprocess --factory all --bus-name org
hyc       34842  34427  0 11月19 ?      00:00:00 /usr/libexec/ibus-engine-simple
root      35214      1  0 11月19 ?      00:00:03 /usr/libexec/fwupd/fwupd
geoclue   35280      1  0 11月19 ?      00:00:02 /usr/libexec/geoclue -t 5
hyc       35637  34230  0 11月19 ?      00:00:00 /usr/libexec/gvfsd-network --spawner :1.3 /org/gtk/gvfs/exec_spaw/7
hyc       35744  34230  0 11月19 ?      00:00:00 /usr/libexec/gvfsd-dnssd --spawner :1.3 /org/gtk/gvfs/exec_spaw/25
root      62162   7383  0 21:00 ?        00:00:00 sshd: root@pts/0
root      62167  62162  0 21:00 pts/0    00:00:00 -bash
```

- UID:用户 ID 
- PID:进程 ID
- PPID:父进程 ID
- C: CPU 用于计算执行优先级的因子。数值越大，表明进程是 CPU 密集型运算，执行优先级会降低;数值越小，表明进程是 I/O 密集型运算，执行优先级会提高
- STIME:进程启动的时间 
- TTY:完整的终端名称 
- TIME:CPU 时间 
- CMD:启动进程所用的命令和参数



## pstree命令

查看进程树

-p :显示进程的 PID
-u :显示进程的所属用户





# 9. 服务管理

## service 指令

在CentOS7.0后很多服务不再使用service,而是`systemctl`. 

**基本语法:** service 服务名 [start | stop | restart | reload | status]

service 指令管理的服务在 /etc/init.d 查看

```shell
[root@centos7 ~]# ll /etc/init.d/
总用量 88
-rw-r--r--. 1 root root 18281 8月  24 2018 functions
-rwxr-xr-x. 1 root root  4569 8月  24 2018 netconsole
-rwxr-xr-x. 1 root root  7923 8月  24 2018 network
-rw-r--r--. 1 root root  1160 10月 31 2018 README
-rwxr-xr-x. 1 root root 45702 10月 11 23:43 vmware-tools
```



## systemctl指令

CentOS7.0后由systemctl管理服务.

**基本语法:** systemctl [start | stop | restart | status] 服务名

systemctl 指令管理的服务在 /usr/lib/systemd/system 查看

```shell
[root@centos7 ~]# ll /usr/lib/systemd/system
总用量 1568
-rw-r--r--. 1 root root  275 11月 14 2018 abrt-ccpp.service
-rw-r--r--. 1 root root  380 11月 14 2018 abrtd.service
-rw-r--r--. 1 root root  361 11月 14 2018 abrt-oops.service
-rw-r--r--. 1 root root  266 11月 14 2018 abrt-pstoreoops.service
-rw-r--r--. 1 root root  262 11月 14 2018 abrt-vmcore.service
-rw-r--r--. 1 root root  311 11月 14 2018 abrt-xorg.service
-rw-r--r--. 1 root root  729 10月 31 2018 accounts-daemon.service
...
-rw-r--r--. 1 root root  657 10月 31 2018 firewalld.service
...
```

其中, firewalld.service是完全服务名, 简称firewalld, 可以直接使用简称, 例如`systemctl status firewalld`

```shell
[root@centos7 ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since 四 2023-11-09 00:11:58 CST; 1 weeks 4 days ago
     Docs: man:firewalld(1)
 Main PID: 6681 (firewalld)
    Tasks: 2
   CGroup: /system.slice/firewalld.service
           └─6681 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

11月 09 00:11:57 centos7.6-study systemd[1]: Starting firewalld - dynamic firewall daemon...
11月 09 00:11:58 centos7.6-study systemd[1]: Started firewalld - dynamic firewall daemon.
```



**systemctl 设置服务的自启动状态**

- systemctl list-unit-files, 查看服务开机启动状态
- systemctl enable 服务名, 设置服务开机启动)
- systemctl disable 服务名, 关闭服务开机启动)
- systemctl is-enabled 服务名, 查询某个服务是否是自启动的



# 10.  监控

## top进程监控

top命令可以实时监控正在执行的进程

**语法**:

​	top [option]

**常用option**

- -d 秒数: 以指定频率刷新数据, 默认3秒
- -i: 不显示闲置或僵尸进程
- -p: 监控指定进程

**在top的界面交互**

- P: 以CPU使用率排序, 默认模式

- M: 以内存使用率排序

- N: 以pid排序

- q: 退出

  

## netstat网络监控

**语法**:

​	netstat [option]

**常用option**

- -a: all, 显示所有socket连接
- -n: 进制使用域名解析功能。链接以数字形式展示(IP地址)，而不是通过主机名或域名形式展示 
- -p: 与链接相关程序名和进程的PID
- -t：所有的 tcp 协议的端口
- -x：所有的 unix 协议的端口
- -u：所有的 udp 协议的端口
- -l：显示所有监听的端口

```shell
[root@centos7 ~]# netstat -anpl | more
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1/systemd           
tcp        0      0 0.0.0.0:6000            0.0.0.0:*               LISTEN      7579/X              
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      7595/dnsmasq        
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      7383/sshd           
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      7380/cupsd          
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      7684/master         
tcp        0      0 172.16.255.160:22       172.16.255.1:64290      ESTABLISHED 117741/sshd: root@p 
tcp        0      0 172.16.255.160:22       172.16.255.1:65514      ESTABLISHED 118643/sshd: root@p 
tcp        0      0 172.16.255.160:22       172.16.255.1:64289      ESTABLISHED 117740/sshd: root@p 
tcp        0      0 172.16.255.160:22       172.16.255.1:64288      ESTABLISHED 117739/sshd: root@p 
tcp6       0      0 :::111                  :::*                    LISTEN      1/systemd           
tcp6       0      0 :::6000                 :::*                    LISTEN      7579/X              
tcp6       0      0 :::22                   :::*                    LISTEN      7383/sshd           
tcp6       0      0 ::1:631                 :::*                    LISTEN      7380/cupsd          
tcp6       0      0 ::1:25                  :::*                    LISTEN      7684/master         
udp        0      0 192.168.122.1:53        0.0.0.0:*                           7595/dnsmasq        
udp        0      0 0.0.0.0:67              0.0.0.0:*                           7595/dnsmasq        
udp        0      0 0.0.0.0:111             0.0.0.0:*                           1/systemd           
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           6590/avahi-daemon:  
udp        0      0 127.0.0.1:323           0.0.0.0:*                           6609/chronyd        
udp        0      0 0.0.0.0:55684           0.0.0.0:*                           6590/avahi-daemon:  
udp        0      0 0.0.0.0:838             0.0.0.0:*                           6615/rpcbind        
udp6       0      0 :::111                  :::*                                1/systemd           
udp6       0      0 ::1:323                 :::*                                6609/chronyd        
udp6       0      0 :::838                  :::*                                6615/rpcbind        
raw6       0      0 :::58                   :::*                    7           6738/NetworkManager 
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node   PID/Program name     Path
unix  2      [ ACC ]     STREAM     LISTENING     259085   34152/gnome-keyring  /run/user/1000/keyring/pkcs11
unix  2      [ ]         DGRAM                    20512    1/systemd            /run/systemd/shutdownd
unix  2      [ ACC ]     STREAM     LISTENING     45427    7579/X               @/tmp/.X11-unix/X0
unix  2      [ ACC ]     STREAM     LISTENING     259382   34427/ibus-daemon    @/tmp/dbus-ncpVaG53
unix  2      [ ACC ]     STREAM     LISTENING     258272   34167/dbus-daemon    @/tmp/dbus-Fn2Z5leh2H

```



# 11. 程序包下载

## rpm管理

全称RedHat Package Manager, 用于互联网下载包的打包及安装工具.

**rpm 包名基本格式**

```shell
[root@centos7 ~]# rpm -q firefox
firefox-60.2.2-1.el7.centos.x86_64
```

- 名称: firefox
- 版本号: 60.2.2-1
- 适用操作系统: el7.centos.x86_64
  - 表示 centos7.x 的 64 位系统
  - 如果是 i686、i386 表示 32 位系统，
  - noarch 表示通用

**语法**

​	rpm [option] [参数]

**常用option**

- -q `firefox`, 查询是否安装firefox包
- -qa, 查询全部rpm安装的软件
- -qi `firefox`, 显示firefox包详细信息
- -ql `firefox`, 显示firefox包的所有文件
- -qf `文件全路径名`, 查询文件所属的软件包

```shell
[root@centos7 ~]# rpm -ql firefox
/etc/firefox
/etc/firefox/pref
/usr/bin/firefox
/usr/lib64/firefox
/usr/lib64/firefox/LICENSE
/usr/lib64/firefox/application.ini
/usr/lib64/firefox/browser/blocklist.xml
/usr/lib64/firefox/browser/chrome
/usr/lib64/firefox/browser/chrome.manifest
/usr/lib64/firefox/browser/chrome/icons
/usr/lib64/firefox/browser/chrome/icons/default
/usr/lib64/firefox/browser/chrome/icons/default/default128.png
/usr/lib64/firefox/browser/chrome/icons/default/default16.png
...
[root@centos7 ~]# rpm -qf /usr/bin/firefox
firefox-60.2.2-1.el7.centos.x86_64
```

- rpm -e `firefox`,  卸载firefox
  - -e: eraser, 擦除
  - --nodeps 不验证包依赖, 强行删除

- rpm -ivh `完整rpm包路径`, 安装rpm包
  - -i: install
  - -v: verbos, 提示
  - -h: hash, 软件包安装的时候列出哈希标记, 常和 -v 一起使用




## YUM管理

Yum 是一个基于 RPM 的包管理工具，能够从指定的服务器自动下载 RPM 包并且安装，包括所有依赖的软件包。



yum list|grep firefox

yum install firefox

yum erase firefox





# 12. Shell

Shell 是一个命令行解释器，它为用户提供了一个向 Linux 内核发送请求以便运行程序的界面系统级程序。



## 12.1 Shell 脚本的执行方式 

### 脚本格式要求

- 脚本以`#!/bin/bash`开头

> `#!`有个专有的名词，叫 `shebang`，发音类似中文的 “蛇棒” 。为什么叫 `shebang` 呢？ 首先 `#` 的英文是 sharp, 而 感叹号 `!` 经常被引用为炸弹，炸弹爆炸就是 bang , 所以 sharp+bang，简读为 `shebang`
>
>后面的 `/bin/bash` 是 Bash Shell 的二进制执行文件路径。是 Unix 类操作系统中最常用的 Shell 程序之一。
>
>所以 `#!/bin/bash` 的作用是：**用于指定默认情况下运行指定脚本的解释器**
>
>当脚本以 `#!/bin/bash` 开头时，内核就知道用 /bin/bash 这个可执行文件来解释并运行这个脚本。
>
>既然是指定一个解释器，那么这个开头就可以根据你指定的解释器有多种不同写法了，比如：
>
>```text
>#!/bin/sh
>#!/bin/bash
>#!/usr/bin/perl
>#!/usr/bin/tcl
>#!/bin/sed -f
>#!/usr/awk -f
>```

- 脚本需要有可执行权限

### 脚本的常用执行方式

- 先赋予脚本**执行权限**再执行脚本, 两种执行方式
  - ./hello.sh执行
  -  /root/shcode/hello.sh 绝对路径执行

- `sh hello.sh`可以直接执行, 不用赋予脚本执行权限



## 12.2 shell变量

Linux Shell 中的变量分为，系统变量和用户自定义变量。

### 系统变量

`$HOME、$PWD、$SHELL、$USER` 等等, `set`命令可以显示当前shell中所有变量.



### 自定义变量

**基本语法**

- 定义变量: 变量名=值
- 撤销变量: unset 变量
- 声明静态变量: readonly 变量，注意: 静态变量不能unset

**定义变量的规则**

- 变量名称可以由字母、数字和下划线组成，但是不能以数字开头。
- 等号两侧不能有空格
- 变量名称一般习惯为大写，这是一个规范

**将命令的返回值赋给变量**

- A=\`date\`(反引号)，运行里面的命令，并把结果返回给变量 A
- A=$(date) 等价于反引号

```shell
#!/bin/bash
#定义变量A
A=100

#输出变量需要加上$
echo A=$A
echo "A=$A"

#撤销变量A
unset A
echo A=$A

#声明静态的变量B
readonly B=100
echo "readonly B=$B"

#unset静态变量会报错
#unset B

#将命令的返回值赋给变量
C=`date`
D=$(date)
echo "C=$C"
echo "D=$D"

# 多行注释
:<<!
多行注释1
多行注释2
echo "多行注释 $D"
!
```



### 设置环境变量

1. 在`/etc/profile`配置文件中增加行:  export 变量名=变量值,  将 shell 变量输出为环境变量/全局变量
2. `source /etc/profile`配置文件, 让修改后的配置信息立即生效
3. echo $变量名



### 位置参数变量

当我们执行一个 shell 脚本时，如果希望获取到命令行的参数信息，就可以使用到位置参数变量
比如 : `./myshell.sh 100 200` , 这个就是一个执行 shell 的命令行，100 200是命令行参数, 可以使用位置参数获取命令行参数

- $n :n 为数字，$0 代表命令本身，$1-$9 代表第一到第九个参数，十以上的参数要用大括号包含，如${10}
- $\*: 这个变量代表命令行中所有的参数, $* 把所有参数合并成一个字符串
- $@: 这个变量也代表命令行中所有的参数，但$@会得到一个字符串参数数组。
- $#: 这个变量代表命令行中所有参数的个数

```shell
[root@centos7 ~]# cat positionVar.sh 
#!/bin/bash
echo "\$0: $0, \$1: $1, \$2: $2" 
echo "所有参数字符串\$*: $*"
echo "所有参数数组\$@: $@"
echo "参数个数\$#: $#"

[root@centos7 ~]# sh positionVar.sh 100 200
$0: positionVar.sh, $1: 100, $2: 200
所有参数字符串$*: 100 200
所有参数数组$@: 100 200
参数个数$#: 2
```



### 预定义变量

shell 设计者事先已经定义好的变量，可以直接在 shell 脚本中使用

- $$: 当前进程的进程号PID
- $!: 上一个后台运行命令的进程号PID
- $?: 上一个命令的执行返回状态。 0: 上一个命令正确执行, 非 0: 上一个命令执行不正确

```shell
[root@centos7 ~]# cat preVar.sh 
 #!/bin/bash
echo "当前执行的进程 id=$$" 
#以后台的方式运行一个脚本，并获取他的进程号 
sh  ~/var.sh &
echo "最后一个后台方式运行的进程 id=$!"
echo "执行的结果是=$?"

[root@centos7 ~]# sh preVar.sh 
当前执行的进程 id=13066
最后一个后台方式运行的进程 id=13067
执行的结果是=0
[root@centos7 ~]# A=100
A=100
A=
readonly B=100
C=2024年 01月 03日 星期三 00:01:56 CST
D=2024年 01月 03日 星期三 00:01:56 CST
^C
```



## 12.3 运算符

- $((运算式))
- $[运算式]
- `expr m + n` 
  - //expression 表达式
  - 注意expr运算符间要有空格, `expr m - n`
  - expr  `\*` (乘号有转义字符),  `/` (除), `%` (取余)



## 12.4  条件判断

### 判断语句

在 Shell 中有两种判断格式，分别如下：

```shell
# 写法一
test expression
# 写法二 ,注意括号两端必须有空格
[ expression ]
# 写法三
[[ expression ]]
```

第二种方式相当于第一种的简化。但是第三种形式还支持正则判断，前两种不支持。

判断语句结果为真，`test`命令执行成功（返回值为`0`）；表达式为伪，`test`命令执行失败（返回值为`1`）

使用预定义变量 `$?`可以获取条件判断结果, `0`表示执行成功，非0表示上个命令失败。

```shell
[root@centos7 ~]# test -e var.sh
[root@centos7 ~]# echo $?
0
[root@centos7 ~]# [ -e var.sh ]
[root@centos7 ~]# echo $?
0
[root@centos7 ~]# [ -e var.sh2 ]
[root@centos7 ~]# echo $?       
1
```

### 判断文件状态

- `[ -a file ]`：如果 file 存在，则为`true`。
- `[ -b file ]`：如果 file 存在并且是一个块（设备）文件，则为`true`。
- `[ -c file ]`：如果 file 存在并且是一个字符（设备）文件，则为`true`。
- `[ -d file ]`：如果 file 存在并且是一个目录，则为`true`。
- `[ -e file ]`：如果 file 存在，则为`true`。
- `[ -f file ]`：如果 file 存在并且是一个普通文件，则为`true`。
- `[ -g file ]`：如果 file 存在并且设置了组 ID，则为`true`。
- `[ -G file ]`：如果 file 存在并且属于有效的组 ID，则为`true`。
- `[ -h file ]`：如果 file 存在并且是符号链接，则为`true`。
- `[ -k file ]`：如果 file 存在并且设置了它的“sticky bit”，则为`true`。
- `[ -L file ]`：如果 file 存在并且是一个符号链接，则为`true`。
- `[ -N file ]`：如果 file 存在并且自上次读取后已被修改，则为`true`。
- `[ -O file ]`：如果 file 存在并且属于有效的用户 ID，则为`true`。
- `[ -p file ]`：如果 file 存在并且是一个命名管道，则为`true`。
- `[ -s file ]`：如果 file 存在且其长度大于零，则为`true`。
- `[ -S file ]`：如果 file 存在且是一个网络 socket，则为`true`。
- `[ -t fd ]`：如果 fd 是一个文件描述符，并且重定向到终端，则为`true`。 这可以用来判断是否重定向了标准输入／输出错误。
- `[ -u file ]`：如果 file 存在并且设置了 setuid 位，则为`true`。



**文件权限**

- `[ -r file ]`：如果 file 存在并且可读（当前用户有可读权限），则为`true`。
- `[ -w file ]`：如果 file 存在并且可写（当前用户拥有可写权限），则为`true`。
- `[ -x file ]`：如果 file 存在并且可执行（有效用户有执行／搜索权限），则为`true`。



**文件比较**

- `[ file1 -nt file2 ]`：如果 FILE1 比 FILE2 的更新时间最近，或者 FILE1 存在而 FILE2 不存在，则为`true`。
- `[ file1 -ot file2 ]`：如果 FILE1 比 FILE2 的更新时间更旧，或者 FILE2 存在而 FILE1 不存在，则为`true`。
- `[ FILE1 -ef FILE2 ]`：如果 FILE1 和 FILE2 引用相同的设备和 inode 编号，则为`true`。

```shell
[root@centos7 ~]# cat condFile.sh 
#!/bin/bash

FILE=/root/~/positionVar.sh
# $FILE要放在""之中。这样可以防止$FILE为空，因为这时[ -e ]会判断为真。
# 而放在""之中，返回的就总是一个空字符串，即[ -e "" ]会判断为伪。
if [ -e "$FILE" ]
then echo "file exit"
else echo "file not exit"
fi

[root@centos7 ~]# sh condFile.sh  
file exit
```



### 字符串判断

- `[ string ]`：如果`string`不为空（长度大于0），则判断为真。
- `[ -n string ]`：如果字符串`string`的长度大于零，则判断为真。
- `[ -z string ]`：如果字符串`string`的长度为零，则判断为真。
- `[ string1 = string2 ]`：如果`string1`和`string2`相同，则判断为真。
- `[ string1 == string2 ]` 等同于`[ string1 = string2 ]`。
- `[ string1 != string2 ]`：如果`string1`和`string2`不相同，则判断为真。
- `[ string1 '>' string2 ]`：如果按照字典顺序`string1`排列在`string2`之后，则判断为真。
- `[ string1 '<' string2 ]`：如果按照字典顺序`string1`排列在`string2`之前，则判断为真。



### 整数判断

- `[ integer1 -eq integer2 ]`：如果`integer1`等于`integer2`，则为`true`。
- `[ integer1 -ne integer2 ]`：如果`integer1`不等于`integer2`，则为`true`。
- `[ integer1 -le integer2 ]`：如果`integer1`小于或等于`integer2`，则为`true`。
- `[ integer1 -lt integer2 ]`：如果`integer1`小于`integer2`，则为`true`。
- `[ integer1 -ge integer2 ]`：如果`integer1`大于或等于`integer2`，则为`true`。
- `[ integer1 -gt integer2 ]`：如果`integer1`大于`integer2`，则为`true`。



### 正则判断

`[[ expression ]]`判断形式，支持正则表达式。

```
[[ string1 =~ regex ]]
```

上面的语法中，`regex`是一个正则表示式，`=~`是正则比较运算符。

```shell
#!/bin/bash
INT=-5
if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
  echo "INT is an integer."
  exit 0
else
  echo "INT is not an integer." >&2
  exit 1
fi
```



### 多判断语句-逻辑运算

通过逻辑运算，可以把多个`test`判断表达式结合起来，创造更复杂的判断。三种逻辑运算`AND`，`OR`，和`NOT`，都有自己的专用符号。

- `AND`运算：符号`&&`，也可使用参数`-a`。
- `OR`运算：符号`||`，也可使用参数`-o`。
- `NOT`运算：符号`!`。

```shell
#!/bin/bash
MIN_VAL=1
MAX_VAL=100
INT=50
if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
  if [[ $INT -ge $MIN_VAL && $INT -le $MAX_VAL ]]; then
    echo "$INT is within $MIN_VAL to $MAX_VAL."
  else
    echo "$INT is out of range."
  fi
else
  echo "INT is not an integer." >&2
  exit 1
fi
```



### 算术判断((...))

Bash 还提供了`((...))`作为算术条件，进行算术运算的判断。

```shell
if ((3 > 2)); then  
	echo "true"
fi
```

上面代码执行后，会打印出`true`。

注意，算术判断不需要使用`test`命令，而是直接使用`((...))`结构。这个结构的返回值，决定了判断的真伪。

如果算术计算的结果是`非零值`，则表示判断`成立`。这一点跟命令的返回值正好相反，需要小心。

```shell
$ if ((1)); then echo "It is true."; fi
It is true.
$ if ((0)); then echo "It is true."; else echo "it is false."; fi
It is false.
```

上面例子中，`((1))`非0表示判断成立，`((0))`表示判断不成立。

算术条件`((...))`也可以用于变量赋值。

```shell
$ if (( foo = 5 ));then echo "foo is $foo"; 
fifoo is 5
```

上面例子中，`(( foo = 5 ))`完成了两件事情。首先把`5`赋值给变量`foo`，然后根据返回值`5`，判断条件为真。

注意，赋值语句返回等号右边的值，如果返回的是`0`，则判断为假。

```shell
$ if (( foo = 0 ));then echo "It is true.";else echo "It is false."; fi
It is false.
```

下面是用算术条件改写的数值判断脚本。

```shell
#!/bin/bash
INT=-5
if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
  if ((INT == 0)); then
    echo "INT is zero."
  else
    if ((INT < 0)); then
      echo "INT is negative."
    else
      echo "INT is positive."
    fi
    if (( ((INT % 2)) == 0)); then
      echo "INT is even."
    else
      echo "INT is odd."
    fi
  fi
else
  echo "INT is not an integer." >&2
  exit 1
fi
```

只要是算术表达式，都能用于`((...))`语法



## 12.5 流程控制

### if语句

`if`是最常用的条件判断结构，只有符合给定条件时，才会执行指定的命令。

```shell
if commands; then
  commands
[elif commands; then
  commands...]
[else
  commands]
fi

# if和then写在同一行时，需要分号分隔 写成两行则不需要分号。
if commands
then
  commands
[elif commands
then
  commands...]
[else
  commands]
fi

#分号是 Bash 的命令分隔符,多个命令写在一行需要使用分隔符
if true; then echo 'hello world'; fi
```

if语句常跟test命令一起使用

```shell
# 写法一
if test -e /tmp/foo.txt
then
  echo "Found foo.txt"
fi
# 写法二
if [ -e /tmp/foo.txt ]
then
  echo "Found foo.txt"
fi
```



### case 语句

`case`结构用于多值判断，可以为每个值指定对应的命令，跟包含多个`elif`的`if`结构等价.

```shell
case expression in
  pattern )
    commands 
    ;; # 表示结束
  pattern )
    commands ;;
  ...
  * ) 
    # *匹配任意输入
  	commands
esac
```
`expression`是一个表达式，`pattern`是表达式的值或者一个匹配模式.

匹配模式可以使用各种通配符，下面是一些例子。

- `a)`：匹配`a`。
- `a|b)`：匹配`a`或`b`。
- `[[:alpha:]])`：匹配单个字母。
- `???)`：匹配3个字符的单词。
- `*.txt)`：匹配`.txt`结尾。
- `*)`：匹配任意输入，通常作为`case`结构的最后一个模式。

```shell
#!/bin/bash
# -n 取消尾随换行符
echo -n "输入一个1到3之间的数字（包含两端）> "
read character
case $character in
  1 ) 
  	echo 1
    ;;
  2 ) 
  	echo 2
    ;;
  3 ) 
  	echo 3
    ;;
  * ) echo 输入不符合要求
esac
```





### for循环

```shell
[root@centos7 ~]# cat for1.sh 
#!/bin/bash

for item in {1..5}
do
    echo $item;
done
[root@centos7 ~]# sh for1.sh 
1
2
3
4
5
[root@centos7 ~]# cat for3.sh 
#!/bin/bash
for((i=1;i<=5;i++));
do 
    echo $i;
done
[root@centos7 ~]# sh for3.sh 
1
2
3
4
5
```

```shell
[root@centos7~]# cat for2.sh 
#!/bin/bash

for i in `ls`;
do 
    echo $i is file name\! ;
done
[root@centos7 ~]# sh for2.sh 
condFile.sh is file name!
delete.sh is file name!
demo.txt.gz is file name!
folder is file name!
```



### while循环

```shell
[root@centos7 ~]# cat while1.sh 
#!/bin/bash

# 循环输出1-5的数字, 并求和
i=1
sum=0
while [ $i -le 5 ]
do
    echo "i=$i"
    i=$(( $i + 1 ))
    sum=$(($sum+$i))
done
echo $sum
[root@centos7 ~]# sh while1.sh 
i=1
i=2
i=3
i=4
i=5
20
```

```shell
#!/bin/bash
# 死循环
while true
do
    echo 'helloword'
done
```

```shell
[root@centos7 ~]# cat while2.sh 
#!/bin/bash
# 读取doc.txt内容并输出
# 方法1
while read line
do
    echo $line
done <./doc.txt

echo "-------------"

# 方法2
cat ./doc.txt | while read line
do
    echo $line 
done
[root@centos7 ~]# sh while2.sh 
content 1
content 2
content 3
-------------
content 1
content 2
content 3
```



### read命令

read命令可以让脚本在执行过程中接受用户输入的一个变量，方便后面的代码使用。用户按下回车键，就表示输入结束。

```
read [-options] [variable...]
```

上面语法中，`options`是参数选项，`variable`是用来保存输入数值的一个或多个变量名。如果没有提供变量名，环境变量`REPLY`会包含用户输入的一整行数据。

```shell
[root@centos7 ~]# cat read.sh 
#!/bin/bash
echo -n "输入一些文本 > "
read text
echo "你的输入：$text"
[root@centos7 ~]# sh read.sh 
输入一些文本 > 000
你的输入：000
```

`read`可以接受用户输入的多个值。

```shell
[root@centos7 ~]# cat read2.sh 
#!/bin/bash
echo Please, enter your firstname and lastname
read FN LN
echo "Hi! $LN, $FN !"
[root@centos7 ~]# sh read2.sh 
Please, enter your firstname and lastname
Ted Huang
Hi! Huang, Ted !
```

`read`根据用户的输入(空格分割)，同时为两个变量赋值。

如果用户的输入项少于`read`命令给出的变量数目，那么额外的变量值为空。如果用户的输入项多于定义的变量，那么多余的输入项会包含到最后一个变量中。



**`read`命令还可以用来读取文件**

```shell
while read myline
do
  echo "$myline"
done < $filename
```



**[option]选项**

- `-t` 设置了超时的秒数。如果超过了指定时间，用户仍然没有输入，脚本将放弃等待，继续向下执行。

```shell
#!/bin/bash
echo -n "输入一些文本 > "
if read -t 3 response; then
  echo "用户已经输入了"
else
  echo "用户没有输入"
fi
```

- `-p`指定用户输入的提示信息,   等效于先echo再read

```shell
read -p "Enter one or more values > " text
echo "type: $text"
```

- `-a`把用户的输入赋值给一个数组，从零号位置开始。

```shell
[root@centos7 ~]# read -a array
spring mybatis dubbo
[root@centos7 ~]# echo ${array[2]} 
dubbo
```

- `-n `只读取若干个字符作为变量值，而不是整行读取。

```shell
[root@centos7 ~]# read -n 3 letter
abcdefghij
[root@centos7 ~]# echo $letter
abc
```

- `-e`参数允许用户输入的时候，使用`readline`库(行操作)提供的快捷键，比如tab自动补全。

```shell
[root@centos7 ~]# cat readline.sh 
#!/bin/bash
echo Please input the path to the file:
read -e fileName
echo $fileName
[root@centos7 ~]# sh readline.sh 
Please input the path to the file:
/root/~/   # 允许使用tab自动提示补全
3               condFile.sh     folder/         if.sh           preVar.sh       read3.sh        read.sh         
5               demo.txt.gz     if2.sh          positionVar.sh  read2.sh        readline.sh     var.sh 
```

- `-d delimiter`：定义字符串`delimiter`的第一个字符作为用户输入的结束，而不是一个换行符。
- `-r`：raw 模式，表示不把用户输入的反斜杠字符解释为转义字符。
- `-s`：使得用户的输入不显示在屏幕上，这常常用于输入密码或保密信息。
- `-u fd`：使用文件描述符`fd`作为输入。



## 12.6 函数

### 系统函数

统函数为 Linux 自带的函数, 可以在 shell 编写中直接使用



**`basename`函数**
用于获取文件名, 根据给出的文件路径截取出文件名

语法:
`basename [pathname] [suffix]`

- 根据根据指定字符串或路径名进行截取文件名, 比如: 根据路径"/root/shells/aa.txt", 可以截取出 aa.txt；
- `suffix`: 用于截取的时候去掉指定的后缀名；

```shell
[root@centos7 ~]# basename /root/base/e.txt
e.txt
[root@centos7 ~]# basename /root/base/e.txt .txt
e
```



**`dirname`函数**

从指定文件的绝对路径, 去除文件名，返回剩下的前缀目录路径
语法:
`dirname [pathname]`

```shell
[root@centos7 ~]# dirname /root/desktop/for2.sh 
/root/desktop
```



### 自定义函数

**定义函数**

在Shell中定义函数的方法如下：

```shell
function 函数名 {
    命令1
    命令2
    ...
}
```

或者

```shell
函数名 () {
    命令1
    命令2
    ...
}
```



**带参函数**

函数可以接受参数。参数通过`$1`、`$2`、`$3`等变量来引用

```bash
[root@centos7 ~]# cat greet.sh 
#!/bin/bash

function greet {
    echo "Hello, $1, $2!"
}

# 多参数空格隔开
greet Tom, Jan

[root@centos7 ~]# sh greet.sh 
Hello, Tom, Jan!
```

- `$1`~`$9`：函数的第一个到第9个的参数。若多于9个，那么第10个参数用`${10}`的形式引用，以此类推。
- `$0`：函数所在的脚本名。
- `$#`：函数的参数总数。
- `$@`：函数的全部参数，参数之间使用空格分隔。
- `$*`：函数的全部参数，参数之间使用变量`$IFS`值的第一个字符分隔，默认为空格，但是可以自定义。



**获取函数返回值**

```shell
[root@centos7 ~]# cat f2.sh 
#!/bin/bash

function add {
    local sum=$(($1 + $2))
    echo "$sum, hahaha"
    # return后面不跟参数, return等效于return 0。
    return 0
}

# 获取函数输出值
result=$(add 2 3)
echo $result

# 函数执行状态码0成功, 非0失败
echo $?

[root@centos7 ~]# sh f2.sh 
5, hahaha
0
```



**全局变量和局部变量，local 命令**

函数体内直接声明的变量，属于全局变量，整个脚本都可以读取。

函数体内用`local`命令声明局部变量,  函数体外引用不到。

```shell
[root@centos7 root_desktop]# cat fn_var.sh 
#!/bin/bash

fn () {
  local local_foo
  local_foo=1
  echo "fn: local_foo = $local_foo"

  global_foo=2
  echo "fn: global_foo = $global_foo"

}

fn
echo "local_foo=$local_foo"
echo "global_foo=$global_foo"

[root@centos7 root_desktop]# sh fn_var.sh 
fn: local_foo=1
fn: global_foo=2
local_foo=
global_foo=2
```



### 一个完整的自定义tomcat服务管理脚本

```shell
#!/bin/sh

BOOT_PATH=$(cd `dirname $0`; pwd)
BIN_PATH=${BOOT_PATH}/bin
LOG_PATH=${BOOT_PATH}/logs

start(){
    echo ">>>Server starting..."
    rm -f ${LOG_PATH}/*.out
    rm -f ${LOG_PATH}/*.log
    ${BIN_PATH}/startup.sh
    echo ">>>Server started."
}
stop(){
 	echo ">>>Server stopping..."
    ${BIN_PATH}/shutdown.sh
    sleep 2
    stopall
    echo ">>>Server stopped."
}
stopall(){   
    ps -ef|grep ${BIN_PATH}|grep -v "grep"|awk '{print $2}'|while read pid
	do
		kill $pid
	done
	sleep 2
	ps -ef|grep ${BIN_PATH}|grep -v "grep"|awk '{print $2}'|while read pid
	do
		kill -9 $pid
	done
} 

case "$1" in  
  start)  
    start  
    ;;  
  stop)   
    stop
    ;; 
  stopall)  
    echo ">>>Server stopping..."
    stopall
    echo ">>>Server stopped."
    ;;  
  restart)  
    stop  
    sleep 1
    start  
    ;;  
  stat)
    echo "======================"
    echo "Process Info:"
    ps -ef|grep ${BIN_PATH}|grep -v grep
     
    echo ""
    ps -ef|grep ${BIN_PATH}|grep -v grep|awk '{print $2}'|while read pid
	do
	    echo "--------------"
	    echo "Handler QTY:"
		#lsof -n| sort|uniq -c|sort -nr|grep $pid
		lsof -n|awk '{print $2}'|sort|uniq -c|sort -nr|grep $pid
		
		echo ""
		echo "Socket QTY:"
		ls /proc/$pid/fd -l|grep socket:|wc -l
		echo "---------------"
	done
  	;; 
  port)
    netstat -tunlp |grep $2
  ;;
  log)
  	echo "================================="
  	tail ${LOG_PATH}/catalina.out -n 500
  	echo "================================="
  	;;
  *)  
    echo "Usage: {start|stop|restart|stat|log|port}"  
    ;;  
esac

exit 
```



# 13 日志

系统日志文件通常保存在 /var/log 目录下。

| 日志文件              | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| **/var/log/boot.log** | 系统启动日志                                                 |
| **/var/log/cron**     | 记录与系统定时任务相关的日志                                 |
| /var/log/cups/        | 记录打印信息的日志                                           |
| /var/log/dmesg        | 记录了系统在开机时内核自检的信息。也可以使用 dmesg 命令直接查看内核自检信息 |
| /var/log/btmp         | 记录错误登录的日志。这个文件是二进制文件，不能直接用 Vi 查看，而要使用 lastb 命令查看 |
| **/var/log/lastlog**  | 记录系统中所有用户最后一次的登录时间的日志。这个文件也是二进制文件，不能直接用 Vi 查看。而要使用 lastlog 命令查看 |
| **/var/log/maillog**  | 记录邮件信息的日志                                           |
| **/var/log/messages** | 它是**核心系统日志文件**，其中包含了系统启动时的引导信息，以及系统运行时的其他状态消息。I/O 错误、网络错误和其他系统错误都会记录到此文件中。其他信息，比如某个人的身份切换为 root，已经用户自定义安装软件的日志，也会在这里列出 |
| /var/log/secure       | 记录验证和授权方面的信息，只要涉及账户和密码的程序都会记录，比如系统的登录、ssh 的登录、su 切换用户，sudo 授权，甚至添加用户和修改用户密码都会记录在这个日志文件中 |
| /var/log/wtmp         | 永久记录所有用户的登录、注销信息，同时记录系统的启动、重启、关机事件。同样，这个文件也是二进制文件。无法直接查看, 要**使用 last 命令查看** |
| **/var/run/utmp**     | 记录当前已经登录的用户的信息。这个文件会随着用户的登录和注销而不断变化，只记录当前登录用户的信息。无法直接查看,  要**使用 w、who、users 等命令查看** |



## 13.1 rsyslogd日志服务

rsyslogd 是一个用于在 Linux 系统上收集、处理和转发日志的系统守护进程。它是目前较为流行的 Linux 日志管理工具之一，取代了Linux6的 syslogd 工具。rsyslogd 是一个强大而灵活的 Linux 系统日志管理工具，可以帮助系统管理员更好地管理和维护系统日志信息。

rsyslogd 可以接收来自不同源头(包括系统内核、应用程序、网络设备和远程主机)的日志信息，并将这些信息按照管理员配置的规则进行分类、过滤、格式化、存储和转发到其他系统中进行集中管理。

rsyslogd 还支持日志信息的压缩、加密和数字签名等安全性功能，确保收集和传输的日志信息的完整性和机密性。



**检查命令**

```shell
# 检查 rsyslogd 服务是否正在运行
systemctl status rsyslog
# 检查是否开机自启动
systemctl is-enabled rsyslog
# 设置开机自启动
systemctl enable rsyslog
```



**日志配置文件`/etc/rsyslog.conf`**

rsyslogd 服务是依赖其配置文件 /etc/rsyslog.conf 来确定哪个服务的什么等级的日志信息会被记录在哪个位置.



**日志格式**

只要是由日志服务 rsyslogd 记录的日志文件，格式就都是一样的。

日志文件的格式包含以下 4 列：

- 事件产生的时间。
- 产生事件的服务器的主机名。
- 产生事件的服务名或程序名。
- 事件的具体信息。

```shell
[root@centos7 ~]# tail -n 2 /var/log/secure
Jan 15 18:30:09 centos7 sshd[79389]: pam_unix(sshd:session): session opened for user root by (uid=0)
Jan 15 18:30:09 centos7 sshd[79388]: pam_unix(sshd:session): session opened for user root by (uid=0)
```



## 13.2 日志轮替

日志轮替就是把旧的日志文件移动并改名，同时建立新的空日志文件，当旧日志文件超出保存的范围时就删除。

**logrotate配置文件**

logrotate 是一个 Linux 系统中常用的日志文件管理工具。配置文件位于 /etc/logrotate.conf 和 /etc/logrotate.d/

```shell
[root@centos7 ~]# cat /etc/logrotate.conf 
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp # 建立的新日志文件，权限是 0664 ，所有者是 root ，所属组是 utmp 组
    minsize 1M # 日志文件最小轮替大小是1MB 。轮替时间到但日志没超1MB也不进行日志转储
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```

**配置说明**

- rotate count 表示保留日志文件的个数，超过这个数量就会删除较早的文件；
- weekly|daily|monthly 表示轮转间隔，可以按周、日或月来轮转；
- compress|nocompress 表示是否压缩轮转后的日志文件；
- delaycompress 表示延迟压缩，即等到下一次轮转时再压缩；
- missingok 表示如果日志文件不存在，则忽略它而不报错；
- notifempty 表示如果日志文件为空，则不进行轮转；
- sharedscripts 表示所有轮转操作都在一个脚本里执行，从而避免启动多个进程；
- postrotate command 表示在轮转完成后执行的命令，例如重新启动相关服务。



**自定义日志的方式**(`推荐第二种`)

- 在 /etc/logrotate.conf 配置文件中写入该日志的轮替策略
- 在 /etc/logrotate.d/ 目录中新建立该日志的轮替文件。





## 13.3 journalctl 内存日志

journalctl 收集的日志文件数以千行计，而且不断更新每次开机、每个事件,  需要使用基本命令进行过滤.

journalctl查看的是内存日志, 因此重启系统后会清空.

journalctl 的配置文件`/etc/systemd/journald.conf`, 不建议修改



**常用命令**

- `journalctl -f`, 滚动查看最新
- `journalctl -n 3`, 查看最新3条
- `journalctl --since 19:00 --until 20:00`,  指定起始时间到结束时间的日志
- `journalctl -o verbose`, 显示详细信息
- `journalctl -p 0`, 按照日志优先级查看
  - `0: 紧急情况`
  - `1: 警报`
  - `2: 危急`
  - `3: 错误`
  - `4: 警告`
  - `5: 通知`
  - `6: 信息`
  - `7：调试`





# 20. bt宝塔-Linux可视化管理系统

https://www.bt.cn/new/index.html



安装

```
yum install -y wget && wget -O install.sh https://download.bt.cn/install/install_6.0.sh && sh install.sh ed8484bec
```

安装成功后控制台会显示登录地址，账户密码

```shell
==================================================================
Congratulations! Installed successfully!
========================面板账户登录信息==========================

 外网面板地址: https://xxx.xxx.xxx.xxx:29686/3dda999b
 内网面板地址: https://172.16.255.160:29686/3dda999b
 username: 8hvhnyqy
 password: c9d235c4
```

宝塔还支持bt命令

```shell
[root@centos7 ~]# bt
==================================宝塔面板命令行====================================
(1) 重启面板服务                  (8) 改面板端口                                   |
(2) 停止面板服务                  (9) 清除面板缓存                                 |
(3) 启动面板服务                  (10) 清除登录限制                                |
(4) 重载面板服务                  (11) 设置是否开启IP + User-Agent验证             |
(5) 修改面板密码                  (12) 取消域名绑定限制                            |
(6) 修改面板用户名                (13) 取消IP访问限制                              |
(7) 强制修改MySQL密码             (14) 查看面板默认信息                            |
(22) 显示面板错误日志             (15) 清理系统垃圾                                |
(23) 关闭BasicAuth认证            (16) 修复面板(检查错误并更新面板文件到最新版)    |
(24) 关闭动态口令认证             (17) 设置日志切割是否压缩                        |
(25) 设置是否保存文件历史副本     (18) 设置是否自动备份面板                        |
(26) 关闭面板ssl                  (19) 关闭面板登录地区限制                        |
(28) 修改面板安全入口             (29) 取消访问设备验证                            |
(0) 取消                                                                           |
====================================================================================
```



