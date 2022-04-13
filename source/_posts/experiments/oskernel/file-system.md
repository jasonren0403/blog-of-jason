---
title: 文件系统实验 date: 2019-11-11 11:53:46 tags: ["OS","Linux","文件系统"]
categories:

- ["技术"]
- ["实验","操作系统内核"]
- ["OS","Linux"]
- ["OS","文件系统"]
  comments: true

---

## 实验目的

1. 掌握文件系统的工作机理，通过一个简单内核源码理解文件系统的主要数据结构。
2. 学习较为复杂的Linux系统下的编程
3. 了解EXT2文件系统的结构

<!--more-->

## 步骤

{% note info no-icon %} 以下操作都是在root用户下进行的，所以不用加`sudo`
{% endnote %}

1. 解压`filesystem2.tar.gz`文件，得到源文件如下图所示 {% asset_img 1.png %}

2. 直接运行`make`，发现报错 {% asset_img 2.png %}

3. 开始解决问题，先在系统中寻找`3.10.0-327.el7.x86_64/build`的链接。 {% asset_img 3.png %}

4. 它指向`/usr/src/kernels/3.10.0-327.el7.x86_64`，再去这个路径里面看看。

{% asset_img 4.png %}

5. 这里什么都没有。说明我的红帽系统缺少相应的内核开发包。

{% asset_img 5.png %}

6. 上网搜索相应的kernel-devel包，下载，并传送到虚拟机上。

7. 使用`rpm2cpio`安装该包。 {% asset_img 6.png %}

8. 再尝试`make`，这次成功。 {% asset_img 7.png %}

9. 使用`insmod aufs.ko`加载内核模块，发现无法安装。再`dmesg | tail`查看内核debug信息。 {% asset_img 8.png %}

10. 方知内核版本不支持，需要重新使用正确的内核版本编译才能通过。内核所需的`rhelversion`为7.5，而我的系统为7.2，显然不匹配。 {% gp 2-2 %} {% asset_img 9.png %} {%
    asset_img 10.png %} {% endgp %}

11. 我选择使用一台新的虚拟机来完成本实验。当然，系统安装结束后，要装载安装光盘镜像，从`Packages`目录中安装内核相关包，`gcc`和`g++`也要顺手安装。 {% gp 2-2 %} {% asset_img 11.png %}
    {% asset_img 12.png %} {% endgp %}

12. 再次`make`。警告不影响编译通过。 {% asset_img 13.png %}

13. 再次尝试加载模块，这次就没有错误出现了。 {% asset_img 14.png %} {% note info %} 内核信息也可以看到加载内核的日志消息。`super.c`
    中`static int __init aufs_init(void)`得以执行 {% asset_img 15.png %} {% endnote %} {% note warning %} 解决module
    verification failed的方法是上网搜的，source:(https://blog.csdn.net/lyw13522476337/article/details/79486326)

但是在加载`mkfs.aufs`后不需解决这个问题 {% endnote %}

14. 直接使用`mount -o loop -t aufs ./image ./dir`挂载该模块，系统会重启。但是在重启后并没有找到刚才挂载上的目录。查阅资料后发现`mount`挂载是临时的，需要修改`/etc/fstab`
    文件，开机的时候，系统就是根据这个分区来挂载系统的。 {% note info %} 我没有选择修改`/etc/fstab`文件，因为这个操作{% label danger@容易使系统崩溃 %} {% endnote %}

15. 格式化分区（关键的一步）

```bash
./mkfs.aufs ./image
```

{% asset_img 16.png %} 然后再`mount`，再次挂载系统就不会重启了。

```bash
insmod aufs.ko
mount -o loop -t aufs ./image ./dir
```

{% asset_img 17.png %}

{% note info no-icon %} 此时`dmesg`的输出如下

```
[ 2082.575699] loop: module loaded
[ 2093.191691] create inode cache success
[ 2093.191698] register filesystem success
[ 2093.191700] aufs module loaded
[ 2094.725812] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725832] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725839] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725846] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725852] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725859] Buffer I/O error on dev loop0, logical block 0, async page read
[ 2094.725874] Buffer I/O error on dev loop0, logical block 3, async page read
[ 2094.756612] aufs_super_block_fill 320017171
[ 2094.756615] now magic number 320017171
[ 2094.756617] aufs super block info:
	magic           = 320017171
	inode blocks    = 1
	block size      = 4096
	root inode      = 1
	inodes in block = 128
[ 2094.756621] now device block size 4096
[ 2094.756633] aufs reads inode 1 from 3 block with offset 32
[ 2094.756642] aufs inode 1 info:
	size   = 0
	block  = 4
	blocks = 1
	uid    = 0
	gid    = 0
	mode   = 40755
[ 2094.756778] aufs_inode_get success
[ 2094.756782] d_make_root success
[ 2094.756783] aufs mounted
```

从debug输出看，aufs文件系统注册时（insmod操作时），经历了加载inode，注册文件系统两个过程。在mount的过程中获得了inode信息，建立了文件系统根目录，其中为super block超级块分配了一个魔法数，super
block大小为4096，它含有128块inode。第一块inode的大小为0，有4块block。Device的块大小为4096，偏移值为32，从第一块inode的block3开始读的。Mode的含义是，这是一个目录（0x40000），所有者拥有全部权限（0x700），用户组可读取，可写入（0x40|0x10），其他用户可读取，可写入（0x40|0x10）。
{% endnote %}

可以用`lsmod`命令查看加载的模块列表，刚才的`aufs`在其中。 {% asset_img 19.png %}

16. 刚刚挂载上去的文件系统在桌面上是这样显示的 {% asset_img 20.png %}
17. 卸载模块：`rmmod xxx.ko`
    {% asset_img 18.png %}

`Super.c`中`static void __exit aufs_fini(void)`得以执行

## 实验关键里程碑数据与结果

### 内核源代码结构分析

{% tabs core,1 %}
<!-- tab <code>aufs.h</code> -->
文件系统各种结构的定义，比如block、inode、entry

```c aufs.h
struct aufs_disk_super_block {    //磁盘超级块结构定义
	__be32	dsb_magic;    //魔法数？
	__be32	dsb_block_size;    //每一块的大小
	__be32	dsb_root_inode;    //每一个inode的根节点
	__be32	dsb_inode_blocks;  //每一个超级块有多少块inode节点
};

struct aufs_disk_inode {    //inode结构定义
	__be32	di_first;     //首节点
	__be32	di_blocks;   //节点块数
	__be32	di_size;     //节点大小
	__be32	di_gid;      //节点组号
	__be32	di_uid;     //节点唯一编号
	__be32	di_mode;   //节点模式和访问权限
	__be64	di_ctime;   //节点创建时间
};
struct aufs_disk_dir_entry {  //磁盘目录对应表定义
	char dde_name[AUFS_DDE_MAX_NAME_LEN];  //磁盘名字
	__be32 dde_inode;  //对应了哪个节点
};

struct aufs_super_block {   //aufs超级块定义
	unsigned long asb_magic;   //魔法数？
	unsigned long asb_inode_blocks;   //已使用（？）inode数量
	unsigned long asb_block_size;     //块的大小
	unsigned long asb_root_inode;     //根inode
	unsigned long asb_inodes_in_block; //总共多少块inode
};
......
```

还定义了`inode`的获取、分配和删除操作。
<!-- endtab -->
<!-- tab <code>aufs.mod.c</code> -->
aufs模块定义，内核版本定义等 {% codeblock aufs.mod.c lang:c line_number:false %} MODULE_INFO(srcversion, "7E6AB09FC99FF944E10E236");
MODULE_INFO(rhelversion, "7.5"); {% endcodeblock %}
<!-- endtab -->
<!-- tab <code>dir.c</code> -->
文件系统目录操作定义和实现，目录管理功能
<!-- endtab -->
<!-- tab <code>file.c</code> -->
定义设备操作`device_open`、`device_release`、`device_read`、`device_write`及其实现
<!-- endtab -->
<!-- tab <code>inode.c</code> -->
文件结点操作
<!-- endtab -->
<!-- tab <code>super.c</code> -->
模块的装载与卸载
<!-- endtab -->
{% endtabs %}

## 实验难点与收获

内核的安装比普通软件的安装要更加困难，它需要系统内核版本的匹配和更多的命令。这个实验给了一个7.5的红帽版本的ko内核，一开始并没有意识到版本问题，编译、挂载总是失败。后来解决了版本问题，但是挂载后系统会自动重启，而Linux重启后所有的挂载均会消失，很坑。

一步关键的动作是使用mkfs.aufs来格式化所给定的image文件。

观看内核源代码，发现它十分复杂，很难一次读通，对我来说是个极大的挑战。

## 实验思考

能不能照着这个给定的模块结构，自己写一个Linux内核模块呢？
