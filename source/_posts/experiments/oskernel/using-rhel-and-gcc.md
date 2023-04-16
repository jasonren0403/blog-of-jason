---
title: Red Hat Linux的安装和使用 
date: 2019-09-26 11:34:07 
tags: ["OS","Red Hat Linux","gcc"]
categories:
  - ["OS","Red Hat Linux"]
  - ["技术"]
  - ["实验","操作系统内核"]
comments: true

---

## 实验目的

本次实验主要熟悉了Linux的一个发行版——`Red Hat Linux`的安装及使用。在安装过程中，我也查阅了一些资料，了解到了这个系统的一些细节和常用操作。为之后的进程管理等实验创建好实验环境和平台。

本次还进行了gcc编译器的安装，安装编译器的过程中，我了解了`Red Hat Linux`下软件安装的一般方法——`rpm`，也了解到一些库的安装可能依赖于另外的一些库，从而需要提前安装上其他库。了解`gcc`编译器，作为编译`.c`文件的一个工具，在今后在Linux系统下的C程序编程中会起到至关重要的作用。

<!--more-->

## 步骤

### 安装Red Hat Linux

{% note info %}
#### 本次实验用到的rhel版本
{% asset_img 附2.png 400 "RHEL版本'RHEL版本截图'" %} 
{% endnote %}

为锻炼好日后独立装载操作系统的能力，本次安装使用基于VMware虚拟机的手动安装方法，在网络上下载好rhel7.2版本的光盘镜像，然后使用光盘加载到虚拟机中，修改BIOS为光盘启动优先，以开启手动安装。

{% asset_img 0-1.png %}

配置过程不再详述。此过程其他截图至实验报告3.1节中查看。 安装结束后，重启进入系统。

{% asset_img 0-10.png "系统界面'系统界面截图'"%}

### Linux文件系统

#### Linux目录结构

{% gp 2-2 %} {% asset_img 2-1.png %} {% asset_img 2-2.png %} {% endgp %}

进入系统后，我们可以看到根目录中都有一些什么文件夹。点击桌面的家目录，然后单击左侧的“计算机”，或者借用命令`ls /`就可以看到。其中有`/bin`，`/boot`，`/dev`，`/etc`，`/home`，`/lib`，`/lib64`，`/media`，`/mnt`，`/opt`，`/proc`，`/root`，`/run`，`/sbin`，`/srv`，`/sys`，`/tmp`，`/usr`，`/var`这么多目录，它们的作用简单介绍如下：

{% tabs Linux folders,1 %}
<!-- tab /bin -->
存放二进制可执行文件，Linux系统的常用命令（`ls`、`mkdir`、`cp`等）就在其中。
<!-- endtab -->
<!-- tab /boot -->
存放用于Linux系统引导时使用的各种文件。
<!-- endtab -->
<!-- tab /dev -->
存放设备文件，在Linux下，“万物皆文件”，所有的外部设备对应了`dev`下的一个文件，例如`dev/sdax`对应了系统下的第一块硬盘的第x个分区。
<!-- endtab -->
<!-- tab /etc -->
存放系统管理与配置文件。
<!-- endtab -->
<!-- tab /boot -->
存放用于Linux系统引导时使用的各种文件。
<!-- endtab -->
<!-- tab /home -->
所有非root用户文件的根目录，用户“家目录”为`/home/<用户名>`，可以用`~`表示。
<!-- endtab -->
<!-- tab /lib -->
存放跟文件系统中的程序运行所需要的共享库及内核模块。共享库又叫动态链接共享库，作用类似windows里的`.dll`文件，存放了根文件系统程序运行所需的共享文件。
{% note info %}
RedHat 中 `/lib` 里面是 32位的库，`/lib64` 里面是 64位的库。
{% endnote %}
<!-- endtab -->
<!-- tab /media -->
是挂载多媒体设备的目录，如默认情况下的光盘、优盘、硬盘等设备都挂载在此目录。
<!-- endtab -->
<!-- tab /mnt -->
一般挂载镜像和硬盘一类的目录。
<!-- endtab -->
<!-- tab /opt -->
额外安装的可选应用程序包所放置的位置。
<!-- endtab -->
<!-- tab /proc -->
虚拟文件系统目录，是系统内存的映射。可直接访问这个目录来获取系统信息。
<!-- endtab -->
<!-- tab /root -->
系统超级管理员的用户主目录。
<!-- endtab -->
<!-- tab /run -->
内有很多pid文件，表示当前系统运行中的进程。pid文件中存储了进程号，可以用于`kill`等操作。
<!-- endtab -->
<!-- tab /sbin -->
superuser binary，存放系统管理员所使用的管理程序。
<!-- endtab -->
<!-- tab /srv -->
主要用来存储本机或本服务器提供的服务或数据。
<!-- endtab -->
<!-- tab /sys -->
包括系统所有的硬件信息以及内核模块等信息。
<!-- endtab -->
<!-- tab /tmp -->
系统产生临时文件的存放目录。
<!-- endtab -->
<!-- tab /usr -->
usr是Unix Software Resource的缩写， 也就是Unix操作系统。软件资源所放置的目录，而不是用户的数据；所有系统默认的软件都会放置到`/usr`。
<!-- endtab -->
<!-- tab /var -->
`/var` 包含系统一般运行时要改变的数据。通常这些数据所在的目录的大小是要经常变化或扩充的。原来`/var` 目录中有些内容是在`/usr`中的，但为了保持/usr目录的相对稳定，就把那些需要经常改变的目录放到 `/var` 中了。
<!-- endtab -->
{% endtabs %}

#### Linux目录功能

{% note info %} 
提前在`mnt`目录下创建好了`usb`文件夹。
{% endnote %} 

向虚拟机插入U盘后，`fdisk -l`有了一些额外的输出，是这样的 

{% gp 2-2 %} 
{% asset_img 3-2.png %} 
{% asset_img 3-3.png %} 
{% endgp %} 

其中，`dev/sdb`应该就是插入的U盘。 

{% note info %} 
实际上，此时操作系统中已经可以看到这个U盘了。
{% asset_img 3-4.png %}
{% endnote %}

{% note warning %}
再运行`mount`命令已经没有作用了
{% asset_img 3-1.png %}
{% endnote %}

可以看到，它实际上已经挂载到了`/run/media/root/设备号`文件夹下。桌面上已经有了盘符显示。

再运行`unmount`命令卸载U盘，桌面移动盘图标会消失。 

{% gp 2-2 %}
{% asset_img 3-5-2.png %}
{% asset_img 3-5-1.png %}
{% endgp %}

#### Linux系统基本命令行操作

{% tabs Linux Commands,1 %}
<!-- tab 关机与重启shutdown -->
{% note primary %}
格式：`shutdown [option] time [warning]`
{% endnote %} 
{% tabs shutdown_and_restart %}
<!-- tab 立即关机-h -->

```bash
shutdown now
shutdown -h now
```

{% note info %}
除了`shutdown`以外，`halt`、`poweroff`、`init 0`也可以关机。
{% endnote %}
<!-- endtab -->

<!-- tab 立即重启-r -->

```bash
shutdown -r now
```

{% note info %}
`init 6`也可以重启。
{% endnote %}
<!-- endtab -->

<!-- tab 延时关机 -->
{% asset_img 4-1.png %}

```bash
shutdown [offset] [message]
```

{% note warning %} 
注意提示为The system is going down for {% label warning @power-off %}... 
{% endnote %}
<!-- endtab -->

<!-- tab 取消延时关机 -->
{% asset_img 4-2.png %}

```bash
shutdown -c
```

{% note warning %}
注意提示The system shutdown
{% label warning @has been cancelled %} ...
{% endnote %}
<!-- endtab -->

<!-- tab 后台提醒重启 -->

```bash
shutdown -r [time] [isBackground]
```

{% gp 2-2 %} {% asset_img 4-3.png %} {% asset_img 4-4.png %} {% endgp %}

{% note info %}
* `&`表示将当前命令放在后台运行，命令行中可能还要做一些工作。（防止终端被卡住）
* `-r`表示reboot，重启系统。 
{% endnote %} 

{% note warning %}
注意提示是The system is going down for {% label warning @reboot %} ...

此时如果打开其他终端，重启警告信息会一分钟发送一次。 
{% endnote %}
<!-- endtab -->

<!-- tab 警告提示次数 -->
{% asset_img 4-5.png %}

{% note info %}
`-k <int>`表示警告信息提示次数，提示完毕后，系统将关机。警告信息提示频率默认为1分钟1次。
{% endnote %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 文件与目录操作 -->
{% tabs files,1 %}
<!-- tab ls -->
{% note primary %} 
格式：`ls [option] [path]`

ls命令一般用于列出Linux中文件夹下的文件列表。 
{% endnote %}

{% tabs ls,1 %}
<!-- tab 默认情况 -->
{% asset_img 5-0.png %} 在任何参数都不加的情况下，输出当前目录的非隐藏文件列表。
<!-- endtab -->

<!-- tab -l参数 -->
{% asset_img 5-1.png %}

`-l`
参数使得输出结果以列表形式展示。最上端的“总用量”代表当前目录下所有文件所占用的空间总和，下方输出，从左到右分别代表文件属性（文件类型+权限属性）、硬链接数（指向该文件的链接数）、文件（目录）拥有者、文件（目录）拥有者所在的用户组、文件所占用的空间（以字节为单位）、文件（目录）最近访问（修改）时间、文件名（如果是一个符号链接，那么会有一个 “→” 箭头符号，后面跟一个它指向的文件名）。
<!-- endtab -->

<!-- tab -d参数 -->
{% asset_img 5-2.png %}

`-d`参数代表只显示目录。
<!-- endtab-->

<!-- tab -a参数-->
{% asset_img 5-3.png %}

`-a`表示显示隐藏文件，在Linux系统下，认为以英文句点(.)开头的文件名为隐藏文件。 

{% note info %}
* `..`和`.`是特殊的隐藏文件，分别代表上级目录和本级目录。
* `-s`代表显示各个文件的大小，后面的`-S`代表以文件大小对输出结果排序。
* Linux系统下，认为每一个目录“大小”为4kb。 
{% endnote %}

<!-- endtab -->

{% endtabs %}
<!-- endtab -->

<!-- tab cp/mv -->
{% note primary %}
格式：`[cp/mv] [option] [pathfrom] [pathto]`
cp命令一般用于复制文件或目录。mv命令一般用于剪切文件或目录，也可以用于重命名。
{% endnote %} 

{% note default %} 
在实验准备之前，先在data目录下写一个1.txt文件，内容为"welcome to linux!"。 
{% endnote %}

{% tabs mv_and_cp %}
<!-- tab 复制 -->
把它复制到同一个目录下 {% asset_img 5-5.png %}
<!-- endtab -->

<!-- tab 移动 -->
把它移动到文件夹a下面 {% asset_img 5-6.png %}
<!-- endtab -->

<!-- tab -i和-r参数 -->
要把上一步的文件复制到另一个文件夹下 
{% asset_img 5-7.png %} 
-i表示交互提示(interactive)，在覆盖文件前会弹出提示。 
{% note warning %} 
现在我们看到直接复制文件夹是不行的。
{% asset_img 5-8.png %} 
此时需要-r参数，表示递归复制文件夹下的所有内容(recursive)。 
{% asset_img 5-9.png %} 
{% endnote %}
<!-- endtab -->
{% endtabs %}

<!-- endtab -->

<!-- tab rm -->
{% note primary %} 
格式：`rm [option] [dirname|filename]`

`rm`命令一般用于删除文件或目录。 
{% endnote %} 

{% note primary %}
实验环境在`/root/data`文件夹下，文件列表如第一条命令执行结果所示。
{% endnote %}

{% asset_img 5-10.png %} 
可以看到，直接`rm`目录名是无法删掉目录的（加上`-d`选项也只能移除空的目录），而`rm`文件名是可以删除文件的（权限足够的情况下）。

{% asset_img 5-11.png %} 
删除目录要用`-r`选项，实际上，它递归移除了其中的所有文件(recursive)。从上图中也可以看到，目录名、文件名都支持通配符。

`-f`表示强制删除(force)，没有任何提示。删除文件夹时一般与`-r`参数配合，以省掉频繁的确认过程。
{% asset_img 5-12.png %}

<!-- endtab -->

<!-- tab mkdir -->
{% note primary %} 
格式：`mkdir [option] [dirname]`

`mkdir`命令一般用于创建目录。 
{% endnote %}

{% note primary %} 
在进行下一步操作前，`data`文件夹为空 
{% endnote %}

在当前目录下直接创建a、b、c文件夹 {% asset_img 5-13.png %}

试图用同样方法在`a`文件夹下建立`test/t`1两级目录。嗯？不行？创建多级目录时，最好使用`-p`选项，建立必要的父文件夹结构。下右图为创建效果 
{% asset_img 5-14.png %} {% asset_img 5-15.png %}

`-m`指定了当前文件夹的权限属性。这回就涉及到了`ls -l`命令输出的左侧的第一个字段了。实际上，`r:read`就是读权限，用数字4表示，`w:write`就是写权限，用数字2表示，`x:excute`就是执行权限，用数字1表示。第一组`rwx`表示文件拥有者权限，第二组`rwx`表示与拥有者同用户组的权限，最后一组表示其他用户权限。

{% asset_img 5-16.png %} 
{% note success %} 
例如，用于ssh登录的`authorized_keys`文件的权限值为600，只允许当前用户的写和读权限，其他用户没有任何权限，这样做是出于安全的考虑。
{% endnote %}

<!-- endtab -->

<!-- tab rmdir -->
{% note primary %} 
格式：`rmdir [option] [dirname]`

`rmdir`命令一般用于删除空目录。和`rm -d`的作用效果一致 
{% endnote %} 
{% asset_img 5-17.png %}
<!-- endtab -->

<!-- tab cd和pwd -->
{% note primary %} 
格式：`cd [option] dirname`

`cd`用于改变当前工作路径，影响后续命令的执行路径(change directory)。 
{% asset_img 5-18.png %}

`pwd [option]`

`pwd`用于打印出当前工作路径(print working directory)
{% asset_img 5-19.png %}

{% endnote %}
<!-- endtab -->

<!-- tab cat -->
{% note primary %} 
格式：`cat [option] file ... `

`cat`命令一般用于拼接(concatenate)文件或标准输入，并在标准输出中输出结果。 
{% endnote %}

{% note info no-icon 例子%} 
例如：`t1`文件夹下有`a.txt`，`b.txt`文件，现在要把它们打印在控制台上（“直接打开”），结果如下 
{% asset_img 5-20.png %} 
其中`-b`选项表示输出非空行的行号。

也可以将标准输入重定向至文件中，使用 `>` 符号。使用<kbd>ctrl</kbd>+<kbd>c</kbd>退出重定向模式。
{% asset_img 5-21.png %}

或者将两个文件的内容拼在一起，然后把输出结果重定向至第三个文件中。
{% asset_img 5-22.png %}
{% endnote %}
<!-- endtab -->

<!-- tab find -->
{% note primary %} 
格式：`find [dir]... [option]`

`find`命令一般用于查找文件。一般有`-name`和`-type`两种选项 
{% endnote %} 

{% asset_img 5-23.png %}

`-print`用于打印出所有查找结果。 {% asset_img 5-24.png %}

`-user`用于指定查找的用户范围。 {% asset_img 5-25.png %}

`-exec <command> {} \;`用于将查找到的文件执行command命令，比如以下截图命令为删除空文件（大小为0的文件）。
{% asset_img 5-26.png %} 

{% note warning %}
注意：大括号和反斜杠之间有一个空格 
{% endnote %}
<!-- endtab -->

<!-- tab grep -->
{% note primary %} 
格式：`grep [option] [pattern] [file]`

`grep`命令用于从文件中查找给定格式的字符串并打印出来。 
{% endnote %} 

从下图的演示结果来看，`-i`选项为忽略大小写的查找（用例文本没写出这一点来），`-n`开启了行号显示。对于查找大文件的相关字符串有一定帮助。
{% asset_img 5-27.png %}
<!-- endtab -->

<!-- tab more -->
{% note primary %}
格式：`more [option] file [...]`

用于分页显示大文件。类似于`cat`。
{% endnote %} 

在分页显示模式中，按<kbd>Space</kbd>可以前进一页，<kbd>Backspace</kbd>可以后退一页，<kbd>q</kbd>键退出此模式。模式截图如下。
{% asset_img 5-28.png %}

`more`还可以和其他命令配合使用，比如要列出一个目录下的文件，由于内容太多，我们应该用`more`来分页显示

```bash
ls -l /usr/lib | more
```

{% asset_img 5-29.png %}
<!-- endtab -->
{% endtabs %}

<!-- endtab -->

<!-- tab 打包与解包，压缩与解压 -->
{% tabs zipping,1 %}
<!-- tab tar -->

* `-c` 创建新的压缩文件
* `-v` 显示详细的处理信息
* `-f` 要操作的文件名，这里是所有txt文件
* 会打包成`tar`格式 {% asset_img 6-1.png %} 

{% note info %}
#### 怎么生成文件大小反而更大了？
还得调用`-z`参数！（实际调用了gzip程序进行压缩）生成的`.tar.gz`文件体积果然减小了。
{% asset_img 6-2.png %}
{% endnote %}

* `-x`选项用于解压(extract)文件
* `tar`文件至少要用`-xf`选项，`tar.gz`文件至少要用`-zxf`选项。
{% asset_img 6-3.png %}

<!-- endtab -->

<!-- tab gzip/unzip -->
`gzip`命令主要用于`gz`格式的压缩。

`--best`指以最高压缩比进行压缩（也可以用`-9`来指定），同时，有以最快速度进行压缩的选项，为`--fast`或`-1`。 {% asset_img 6-4.png %}

`-l`查看当前所有`gz`压缩文件的信息（包括压缩前后大小，压缩比，原文件名） {% asset_img 6-5.png %}

`-d`选项可以解压`gz`格式的压缩文件。在这里可以验证原来文本文件的大小。 {% asset_img 6-6.png %}

`unzip`主要用于`zip`格式压缩文件的解压。 {% asset_img 6-7.png %}

`-d`选项可以指定解压至哪一个文件夹。而`unzip`的默认处理方式为`-j`，即直接解压至压缩包同级目录下。注意到`inflating`的输出有所不同。 
{% gp 2-2 %} {% asset_img 6-8.png %} {% asset_img 6-9.png %} {% endgp %}

<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab Linux系统管理 -->
{% tabs linux_system_manage,1 %}
<!-- tab 文件权限 -->
{% tabs sysman,1 %}
<!-- tab chmod -->
{% note primary %}
格式：`chmod [who] [操作符] [mode] 文件名`

`chmod`用于修改文件的权限标志位。
{% endnote %}

{% asset_img 7-1.png "为方便测试执行权限，my_ls文件中的内容是：ls ~" %}

{% note info %}
who选项中，g指同组用户(group)，o指其他用户(others)，u指当前用户(user)，a指所有人(all=g+o+u)。
操作符+/-/=分别代表增加权限、减少权限、设定权限为，mode为rwx三种，分别对应读取，写入，执行权限。

{% label info @新创建的文件一般没有执行权限，需要`chmod +x`来添加权限， %}然后在可执行文件所在文件夹下运行。{% label primary @这一点似乎root用户也不例外。 %} 
{% endnote %}

{% gp 3-3 %} {% asset_img 7-2.png %} {% asset_img 7-3.png %} {% asset_img 7-4.png %} {% endgp %}

登录另一个用户尝试读、写、执行，发现均显示“权限不够”，证明文件权限被正确设置为`600`，即其他用户没有任何权限。

同理，也可以使用八进制掩码来设置文件权限，其中r的权值为4，w的权值为2，x的权值为1，分别对应了读、写、执行权限。 {% asset_img 7-5.png %}

<!-- endtab -->
<!-- tab chgrp -->
{% note primary %}
格式：`chgrp [username] [file]`

`chgrp`用于更改文件所属的用户组。
{% endnote %} 

执行后发现`ls -l`的第四列结果（文件所属用户组）发生变化。 {% asset_img 7-6.png %}
<!-- endtab -->

<!-- tab chown -->
{% note primary %}
格式：`chown [options] user[:group] file...`

`chown`用于更改文件属主（和用户组）。
{% endnote %}
执行后根据`group`的指定与否，发现`ls -l`结果的第三列或第三列和第四列发生变化

{% gp 2-2 %} {% asset_img 7-7.png %} {% asset_img 7-8.png %} {% endgp %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 用户权限 -->
{% tabs user,1 %}
<!-- tab passwd -->
{% note primary %}
格式：`passwd <用户名>`

`passwd`命令用于更改一个用户的unix密码，在更改的过程中，发现它有一定的强度要求。**而且输入的密码不会在屏幕上回显。**
{% endnote %}
注销，用新用户登录，输入刚才设置的密码可以进入系统。

{% asset_img 8-1.png %} {% asset_img 8-2.png %}

{% note primary %}
使用`passwd -S <用户名>` 可以打印出目标用户的密码情况。嗯，SHA512算法还是相对来说比较安全的…… {% asset_img 8-3.png %}
{% endnote %}
`passwd -d <用户名>` 可以清除指定用户的密码。 {% asset_img 8-4.png %}
<!-- endtab -->

<!-- tab su -->
`su <用户名>` 可以不注销当前登录态的情况下切换用户，视密码是否为空而提示输入相应用户密码。（当时root有密码，testusr没有密码）

{% gp 2-2 %} {% asset_img 8-5.png %} {% asset_img 8-6.png %} {% endgp %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 系统使用交互 -->
{% tabs sys_interact,1 %}
<!-- tab write -->
write命令用于传讯息给其他使用者。在执行write 用户名 后，把要传输的信息打过去，按<kbd>ctrl</kbd>+<kbd>c</kbd>结束，传输的信息就会显示在那个用户名的终端里。 在这里，我使用了本地的另一台虚拟机Kali Linux，以ssh方式访问rhel机器，来模拟多用户访问的情况。以下为测试效果。

{% gp 2-2 %} {% asset_img 9-1.png "传输给客户端信息" %} {% asset_img 9-2.png "客户端登录" %} {% endgp %}

<!-- endtab -->

<!-- tab mesg -->
使用`mesg n`可以关闭终端机的写入权限（实际上上图的情况是表明输出权限也一并被关闭）。`mesg y`可以重新打开。

{% gp 2-2 %} {% asset_img 9-3.png "write命令" %} {% asset_img 9-4.png "客户端效果" %} {% endgp %}

<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 磁盘管理 -->
{% tabs disk-man,1 %}
<!-- tab free -->
`free`命令用于查看系统中已用空间和可用空间。表头从左到右的含义为总空间，已用空间，空余空间，共享空间，缓存空间，可用空间。第一行为存储区，第二行为交换区。

{% asset_img 10-0.png %}

`-b | -k | -m`表示以字节，千字节，兆字节的方式来显示数据，可读性更强，不过要使可读性最强，可以使用`-h`选项。 {% asset_img 10-00.png %}
<!-- endtab -->

<!-- tab df -->
`df`命令用于显示目前在Linux系统上的文件系统的磁盘使用情况统计。 {% asset_img 10-1.png %}

`-a`选项包含所有的具有 0 Blocks 的文件系统，`-T`选项显示文件系统的形式。

{% gp 2-2 %} {% asset_img 10-2.png %} {% asset_img 10-3.png %} {% endgp %}

`-t`代表限制列出文件系统的类型，由于当前系统没有`ext3`格式的类型，所以会显示“未处理文件系统”。 {% asset_img 10-4.png %}
`-h`使其中输出的值（指容量相关值）更加具有可读性。
<!-- endtab -->

<!-- tab du -->
`du`命令用于显示目录或文件的大小。默认统计当前文件夹大小。 {% asset_img 10-5.png %}

`du –a /` 的结果，输出很长，只截取了后面的部分，最底部的“3283432”为总用量，`-a`选项则列出了所有文件及目录的大小 {% asset_img 10-6.png %}

`-s`选项仅显示总计容量大小，配合`-h`选项可以快速统计某目录的所占空间大小。 {% asset_img 10-7.png %}
<!-- endtab -->

<!-- tab dd -->
`dd`命令用于以指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。经常用于备份设备数据。

由于设备短缺，测试只好使用从文件复制到本地硬盘中。可以看到一些输出。

直接运行`dd`命令是从标准输入传送到标准输出。

{% gp 2-2 %} {% asset_img 11-1.png %} {% asset_img 11-2.png %} {% endgp %}

<!-- endtab -->

<!-- tab fdformat -->
`fdformat`主要用于低阶格式化软盘，对设备要求较高，由于手边设备有限，并不能得出结果。 {% asset_img 12.png %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 进程管理 -->
{% tabs process_man,1 %}
<!-- tab at -->
`at`命令：用于安排临时任务（与周期性任务`crontab`有所不同）

输入`at <time>`后，进入at命令模式，以<kbd>ctrl</kbd>+<kbd>D</kbd>结束任务输入。

`atq`可以用于查看当前任务列表，`atrm <id>`可以删除临时任务

{% asset_img 13-1.png %} {% asset_img 13-2.png %}

比如，上图描述了在14：45分时，列出用户家目录的文件列表，并把它放入`1.txt`文件中。执行效果如下 {% asset_img 13-3.png %}
<!-- endtab -->

<!-- tab bg、fg和& -->
命令执行通常会阻塞住当前的终端，如果命令执行时间很长，而碰巧又想执行别的命令，就会相当麻烦，`&`可以是当前命令转入后台运行，即使关闭当前终端也不会中断其运行。可以使用`bg`查看后台任务，`fg`将后台任务转回终端查看。
{% note info %} 
这里，`my_ls`添加了`sleep`语句，方便查看后台运行效果
{% endnote %}
{% asset_img 13-4.png %}
<!-- endtab -->

<!-- tab who -->
`who` 命令显示关于当前在本地系统上的所有用户的信息。显示以下内容：登录名、tty、登录日期和时间，如果用户是从一个远程机器登录的，那么该机器的主机名也会被显示出来。

{% asset_img 14-1.png %}
`-H`显示表头，`-T`显示 tty 的状态。 {% asset_img 14-2.png %}
<!-- endtab -->

<!-- tab w -->
`w`命令显示目前登入系统的用户信息。执行这项指令可得知目前登入系统的用户有哪些人，以及他们正在执行的程序。（`testusr`没登录，所以这里没有他的信息）
{% asset_img 14-3.png %} {% asset_img 14-4.png %}
<!-- endtab -->

<!-- tab ps -->
`ps`命令列出系统中当前运行的那些进程，是当前那些进程的快照。

{% blockquote %}
STAT字母表示：

* R 运行 runnable (on run queue)
* S 中断 sleeping
* D 不可中断 uninterruptible sleep (usually IO)
* T 停止 traced or stopped
* Z 僵死 a defunct ('zombie') process

{% endblockquote %}

{% asset_img 14-5.png %}

`-u`选择用户名。 {% asset_img 14-6.png %}

`-l`长格式。多了F，wchan，C等字段。 {% asset_img 14-7.png %}

`-a`显示一个终端的所有进程，除了会话引线。 {% asset_img 14-8.png %}

`-x`显示没有控制终端的进程，同时显示各个命令的具体路径。最常见的组合为`-aux`，通常配合管道`grep`来找到要停止的进程。

{% gp 2-2 %} {% asset_img 14-9.png %} {% asset_img 14-10.png %} {% endgp %}
<!-- endtab -->

<!-- tab kill -->
`kill [sig] <pid>`杀死进程，强杀进程一般使用信号`-9(SIGKILL)`，因为其他信号都可以被忽略 {% asset_img 14-11.png %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->
{% endtabs %}
<!-- endtab -->

<!-- tab 其他命令 -->
{% tabs other,1 %}
<!-- tab echo -->
{% asset_img 15.png %}

在终端中回显输出用户的字符串。
<!-- endtab -->
<!-- tab cal -->
{% gp 2-2 %} {% asset_img 16-1.png "无参数，当月日历" %} {% asset_img 16-2.png "-y参数，当年日历" %} {% endgp %}

显示日历。
<!-- endtab -->
<!-- tab date -->
{% asset_img 17.png %}

输出日期和时间。
<!-- endtab -->
<!-- tab clear -->
{% gp 2-2 %} {% asset_img 18-1.png "清空前" %} {% asset_img 18-2.png "清空后" %} {% endgp %}

清空当前终端中的内容（实际上表现为滚动条向下滚动）。
<!-- endtab -->
{% endtabs %}

<!-- endtab -->
{% endtabs %}

#### Vi的基本使用

{% blockquote %}
Linux系统中一般会内置vi作为文本编辑器，用命令`vi xxx`启用后，有三种模式：命令模式（Command mode），输入模式（Insert mode）和底线命令模式（Last line mode）。刚刚进入vi时，是命令模式，任何输入字符不会加入文件中，而是被解析为命令运行，想要进入输入模式，需要按键盘<kbd>i</kbd>键。输入完成后，可按<kbd>ESC</kbd>键退回命令模式。要退出程序，需要按<kbd>:</kbd>键，切换到底线命令模式中，然后输入<kbd>q</kbd>不保存退出或输入<kbd>w</kbd>保存退出。底线命令模式中支持的命令比命令模式要多。
{% endblockquote %} 

下面是一些命令的运行效果截图。

{% gp 8-4 %}
{% asset_img 19-1.png "用vi new.txt创建一个新文件并同时进入编辑模式" %}
{% asset_img 19-2.png "按i键进入编辑模式" %}
{% asset_img 19-3.png "编辑结束，按ESC键退出编辑模式" %}
{% asset_img 19-4.png "按冒号键切换到底线命令模式" %}
{% asset_img 19-5.png "输入wq!保存并退出" %}
{% asset_img 19-6.png "验证文件内容是否被正确保存" %}
{% asset_img 19-7.png "然后再次编辑，但这次退出时使用q选项" %}
{% asset_img 19-8.png "发现新加的内容没有保存"%}
{% endgp %}

#### Linux系统中的gcc编译器

{% blockquote %}
GCC 原名为 GNU C 语言编译器（GNU C Compiler），因为它原本只能处理C语言。GCC 很快地扩展，变得可处理C++。后来又扩展为能够支持更多编程语言，如Fortran、Pascal、Objective-C、Java、Ada、Go以及各类处理器架构上的汇编语言等，所以改名GNU编译器套件（GNU Compiler Collection）。
{% endblockquote %}

安装Linux系统时本应选择预装gcc套件的，但当时并没有考虑到这个需求。还好Red Hat的镜像中有gcc的软件包，仍旧可以安装。安装过程不再详述。

用vi编写一个简单的c程序，保存为`hello.c`。

```c hello.c
/*--hello.c--*/
#include <stdio.h>
int main(){
	printf("Hello world\n");
	return 0;
}
```

使用`gcc`命令编译该源代码。`-o`参数指定生成目标文件，它可以直接执行（有x权限）。

{% tabs using gcc,1 %}
<!-- tab 编译 -->

```bash
gcc hello.c -o hello
```

{% asset_img 20-1.png %}
<!-- endtab -->
<!--tab 执行 -->

```bash
./hello
```

{% asset_img 20-2.png %}
<!-- endtab -->
{% endtabs %}

## 实验关键里程碑数据与结果

### 安装Red Hat Linux

{% gp 3-3 %} {% asset_img 0-2.png %} {% asset_img 0-3.png %} {% asset_img 0-4.png %} {% endgp %}

{% gp 4-3 %} {% asset_img 0-5-1.png %} {% asset_img 0-5-2.png %} {% asset_img 0-6-1.png %} {% asset_img 0-6-2.png %} {% endgp %}

{% gp 4-3 %} {% asset_img 0-7.png %} {% asset_img 0-8.png %} {% asset_img 0-9-1.png %} {% asset_img 0-9-2.png %} {% endgp %}

### 安装gcc编译器

{% asset_img 附3.png %}

## 实验难点与收获

完成本实验花费了个人约8个小时的时间，其中有6个小时的“上机”时间（在系统），剩余时间用于实验报告的写作上。感觉这次的实验还是很简单的，基本上没有难点，除了最后在配置gcc软件包时，需要先安装大量依赖软件包，占用了一些时间。

## 实验思考

* 通过本实验，我感受到了Linux系统相对于Windows的强安全性。
    - 首先，Linux系统在终端中输入任何密码时，都不会有回显，在设置用户密码时，要求它有一定的强度（≥8个字符，且还要通过字典检查）
    - 除了root用户，所有其他用户的根目录均局限于`home`目录下，由于权限码的限制，一旦离开了`home`目录，便很难再拥有权限操作文件，因为往往系统文件的权限值对于其他用户均设置为很低的值，而且Red Hat Linux对于创建的新用户，默认是连sudo权限都没有的（下图，还带着提示），这意味着他们几乎不可能进入系统空间和其他用户的空间，也更不可能列出文件目录或者是执行文件。

    {% asset_img 附1.png '新用户默认没有sudo权限' %}

* Linux系统的命令是相对于Windows系统来说最大的不同点，用惯了图形化界面操作系统的人，总归对冷冰冰的命令行页面下不去手。这次的安装虽然装的是带有GUI界面的服务器版本（下图），但主要的操作还是放到了命令行中，因为可以很直观的看到输出。

    {% asset_img 0-6-2.png 400 "安装时可以选择带有GUI的服务器" %}

* 上手使用了一些Linux中的命令后，感觉还是比较好掌握的，因为它们基本上是一些英文单词的缩写。命令格式也非常好理解，一个短横线往往跟短选项，而两个短横线往往跟长选项。有些短选项是可以合并的（比如`ls -la`等价于 `ls -l -a`）。
