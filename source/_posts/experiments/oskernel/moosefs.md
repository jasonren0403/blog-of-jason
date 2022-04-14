---
title: 分布式文件系统MooseFS搭建 
date: 2019-12-21 13:47:00 
tags: ["OS","MooseFS","文件系统"]
categories:
  - ["OS","分布式文件系统"]
  - ["技术"]
  - ["实验","操作系统内核"]
comments: true

---

## 实验目的

配置分布式文件系统MooseFS，并尝试挂载操作。
<!-- more -->

## 步骤

### 源码编译

根据MooseFS官方Github上的指示，在RHEL上解压源码包，并创建用户，安装好必要的build包 {% gp 2-2 %} {% asset_img 1.png %} {% asset_img 2.png %} {% endgp
%}

### 配置master端

{% note info %} IP：192.168.235.134，运行Red Hat Linux系统 {% endnote %} 创建相关文件夹和用户，运行`make`和`make install`。 {% asset_img
3.png %} {% asset_img 4.png %} {% note info %} 注意mfs需要一个用户名为`mfs`的用户，它属于`mfs`用户组 {% endnote %} {% asset_img 5.png %}
进入mfs安装目录下的`etc/mfs`文件夹，将mfsmaster、msfexports、msftopology的`cfg`文件后的`sample`后缀去掉。然后将`var/mfs`文件夹下的`metadata.mfs.empty`
更名为`metadata.mfs`。删除原来的empty文件。 {% asset_img 6.png %} 做完上述配置，就可以启动mfs安装目录下的`sbin/mfsmaster`了（`mfsmaster start`
）。其在9421端口下监听客户端连接，9420端口下监听chunkserver连接。 {% asset_img 7.png %} 可以开启sbin下的`mfscgiserv`
，以web端方式来查看服务器情况，默认监听端口为9425。需要在`/etc/hosts`中的绑定`mfsmaster`的解析ip。 {% asset_img 8.png %} {% asset_img 9.png %} {%
asset_img 10.png %}

### 配置chunkserver

{% tabs chunkservers,1 %}
<!-- tab A端 -->
{% note info %} IP：192.168.235.146，运行Ubuntu系统 {% endnote %}

1. 添加用户，创建目录，安装必需软件包。 {% asset_img 17.png %} {% asset_img 18.png %}
2. 配置<code>configure</code>，然后运行<code>make && make install</code>。 {% asset_img 22.png %} {% asset_img 23.png %}
3. Chunkserver需要<code>mfschunkserver</code>和<code>mfshdd</code>两个配置文件。将它们更名。 {% asset_img 24.png %}
4. 创建两个挂载用的共享目录（实际上最后只用了第一个） 网上的教程上说最好给每一个目录配备单独的硬盘分区或lvm逻辑卷，但是本次实验不需要。 {% asset_img 25.png %}
5. 在<code>mfshdd.cfg</code>上添加共享目录。 {% asset_img 26.png %}
6. 运行mfs安装目录<code>/sbin/mfschunkserver start</code>启动chunk server。它在9422端口监听。 {% asset_img 27.png %}

<!-- endtab -->
<!-- tab B端 -->
{% note info %} IP：192.168.235.149，运行Red Hat Linux系统 {% endnote %}

1. 下列操作与A端类似 {% gp 4-3 %} {% asset_img 11.png %} {% asset_img 12.png %} {% asset_img 14.png %} {% asset_img 15.png %} {%
   endgp %}
2. 对了，要在chunkserver的<code>/etc/hosts</code>中绑定mfsmaster的地址。A端的ubuntu系统也要这么做 {% asset_img 13.png %}
3. <code>mfschunkserver start</code>启动数据服务器。它在9422端口监听。 {% asset_img 16.png %}

<!-- endtab -->
<!-- tab client -->
{% note info %} IP：192.168.235.148，运行Ubuntu系统 {% endnote %} {% asset_img 19.png %} {% asset_img 20.png %}
<!-- endtab -->
{% endtabs %}

## 实验关键里程碑数据与结果

使用mfs安装目录`/bin/mfsmount 目录名 -H <mfsmaster IP地址>`来挂载目录，在此之前，先依次启用mfsmaster和两个mfschunkserver。显示accepted，即代表挂载成功 {%
asset_img 32-Client.png %} 此时查看被挂载的目录，会发现它空间很大，应该是所有CHUNKSERVER存储空间的和。 {% asset_img 33-df.png %}

查看mfsmaster日志可以看到chunkserver注册连接的记录。 {% asset_img 31-Server.png %} Chunkserver端也有相应的记录。（可以看到，此端成功连接到了主服务器） {% asset_img
30-Bconnected.png %} 实验结束后，先断开所有文件夹的挂载，然后停止chunkserver，最后停止masterserver。 {% tabs stopserver,1 %}
<!-- tab chunkserverA -->
{% asset_img 34-stopchunk1.png %}
<!-- endtab -->
<!-- tab chunkserverB -->
{% asset_img 34-stopchunk2.png %}
<!-- endtab -->
<!-- tab masterserver -->
{% asset_img 34-stopmaster.png %}
<!-- endtab -->
{% endtabs %}

访问master端的cgi网页，可以发现很多有用的信息，如下图所示 {% gp 3-3 %} {% asset_img disks.png "所有分布式存储的挂载目录信息" %} {% asset_img servers.png "
所有服务器的在线情况信息" %} {% asset_img info.png "所有总的Info" %} {% endgp %}

## 实验难点与收获

本次实验的步骤和参考资料来自：
https://www.cnblogs.com/rockbes/p/3989395.html
https://www.cnblogs.com/rockbes/p/3989410.html
https://www.cnblogs.com/rockbes/p/3989419.html
https://github.com/moosefs/moosefs

这是一个相当复杂的配置实验！里面有很多坑点和难点，不过最后还是一步一步做了下来，很有成就感。

收获就是……很多步骤需要一定权限，这时候使用root权限是非常方便的，还有就是，涉及到虚拟机间的网络连接，一定要关防火墙！`systemctl stop firewalld.service`

由于只是练习配置，各个角色之间联通即可，所以很多功能都还没有用上。实际上，MooseFS的配置文件很有东西，潜力丰富。特别是本次实验没怎么用到的日志功能，在恢复受损文件系统上有着非常重要的作用。

## 实验思考

### 坑点集中

1. server start时出现can't create lockfile in working directory: EACCES (Permission denied)
   ，是因为编译时没指定`--with-default-user=mfs`，所以生成的配置文件里运行用户是`nobody`，运行组是`mfs`，将用户修改即可成功。

我采用的是更省事的方法：编译时指定`default-user=mfs`

2. 连不上mfsmaster {% asset_img 28.png %} 检查mfsexports文件中有没有为client IP放开权限。（取消#注释） 前面那个*打头的也最好放开。 {% asset_img 29.png %}
   如果还不行，关闭防火墙。

### 了解MooseFS

MooseFS是一种分布式文件系统，MooseFS文件系统结构包含四种角色。 {% tabs role,1 %}
<!-- tab 管理服务器master -->
负责各个数据存储服务器的管理，文件读写调度，文件空间回收以及恢复、多节点拷贝
<!-- endtab -->
<!-- tab 元数据日志服务器metalogger -->
负责备份 master 服务器的变化日志文件，文件类型为 changelog_ml.*.mfs，以便于在 master server 出问题的时候接替其进行工作
<!-- endtab -->
<!-- tab 数据存储服务器chunkserver -->
听从管理服务器调度，提供存储空间，并为客户提供数据传输。真正存储用户数据的服务器。存储文件时，首先把文件分成块，然后这些块在数据服务器chunkserver之间复制（复制份数可以手工指定，建议设置副本数为
3）。数据服务器可以是多个，并且数量越多，可使用的“磁盘空间”越大，可靠性也越高。
<!-- endtab -->
<!-- tab 客户机挂载使用client -->
客户端挂载远程mfs服务器共享出的存储并使用。 通过fuse内核接口挂载进程管理服务器上所管理的数据存储服务器共享出的硬盘。共享的文件系统的用法和 nfs 相似。使用 MFS 文件系统来存储和访问的主机称为 MFS 的客户端，成功挂接
MFS 文件系统以后，就可以像以前使用 NFS 一样共享这个虚拟性的存储了。
<!-- endtab -->
{% endtabs %} 本次实验用到了角色1、3和4。
