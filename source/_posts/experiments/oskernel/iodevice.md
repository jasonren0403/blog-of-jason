---
title: 设备管理实验 
date: 2019-12-04 13:20:38 
tags: ["OS","Linux","IO(输入输出设备)"]
categories:
  - ["OS","Linux 内核"]
  - ["OS","IO设备"]
  - ["技术"]
  - ["实验","操作系统内核"]
comments: true

---

## 实验目的

尝试加载运行给定的内核程序，以理解内核的结构。尝试修改设备管理相关代码以实现一些简单的功能。
<!-- more -->

## 步骤

### 实验准备

1. 首先解压给定的压缩包。

    ```bash
    tar -zxvf iodevicemanagement.tar.gz
    ```
	
    {% asset_img 1.png %}

2. 安装`build-essential`，发现失败。在

    ```bash
    sudo rm /var/lib/apt/lists/* -vf
    sudo apt-get update
    ```

    后可以解决。接下来安装`qemu`，过程类似，没有截图。 {% asset_img 2.png %}

3. 先对build.sh加执行权限。

    ```bash
    chmod +x tools/build.sh
    ```

4. 然后运行`make`
   {% asset_img 5.png %}
5. Make生成结束后，使用`make start`启动虚拟机。 {% asset_img 6.png %}
6. 出来一个叫做QEMU的黑色窗口，到这里，实验环境算是准备好了。 {% asset_img 7.png %}

### 尝试修改内核代码

{% note info %}
#### 需求
修改内核中I/O设备代码，实现按<kbd>F12</kbd>，`ls`命令显示的信息替换成`*`，再按<kbd>F12</kbd>恢复正常，如此反复。
{% endnote %}

原来，虚拟机系统按<kbd>F12</kbd>时，显示如下信息：
{% asset_img 8-f12.png %}
根据这个输出，我们找到了相应的源代码。
{% codeblock /kernel/sched.c lang:c mark:5,9 %}
void show_task(int nr,struct task_struct * p)
{ 
	int i,j = 4096-sizeof(struct task_struct);

	printk("%d: pid=%d, state=%d, ",nr,p->pid,p->state);
	i=0;
	while (i<j && !((char *)(p+1))[i])
		i++;
	printk("%d (of %d) chars free in kernel stack\n\r",i,j);

}
{% endcodeblock %}

{% note info %}
#### <code>printk</code>是啥？
`printk`是在内核中运行的向控制台输出显示的函数，Linux内核首先在内核空间分配一个静态缓冲区，作为显示用的空间，然后调用`sprintf`，格式化显示字符串，最后调用`tty_write`向终端进行信息的显示。
{% endnote %}

{% tabs change,1 %}
<!-- tab <code>/include/linux/sched.h</code> -->
声明变量`int f12_state`
{% codeblock /include/linux/sched.h lang:c line_number:false %}
int f12_state; //if it is 1,all chars are replaced with *
{% endcodeblock %}
<!-- endtab -->
<!-- tab <code>/kernel/sched.c</code> -->
实现<kbd>f12</kbd>开关功能

```c /kernel/sched.c
f12_state = 0;
void switch_f12(void)
{
    if (f12_state == 1)
        f12_state = 0;
    else
        f12_state = 1;
}
```

<!-- endtab -->
<!-- tab <code>/kernel/chr_drv/console.c</code> -->
修改`con_write`函数，即改变显示的字符。

加入如下代码，在写字符之前根据 `f12_state` 的状态判断是否要将字符修改为 `*`。根据要求，只将字母显示为 `*`。
{% codeblock /kernel/chr_drv/console.c lang:c line_number:false %}
if((c >= 'A' && c <= 'Z' || c >= 'a' && c <= 'z') && f12_state==1) c = '*';
{% endcodeblock %}
<!-- endtab -->
<!-- tab <code>/kernel/chr_drv/keyboard.S</code> -->
在开头添加对<kbd>f12</kbd>扫描码（`0x58`）的检测

```asm /kernel/chr_drv/keyboard.S
....
func:
 cmpb $0x58,%al  # this
 jne continue_func
 pushl %eax
 pushl %ecx
 pushl %edx
 call switch_f12
 popl %edx
 popl %ecx
 popl %eax
 jmp end_func
....
continue_func:
....
```

<!-- endtab -->
{% endtabs %}

{% note warning %}
对内核文件作修改后，需要重新运行`make`和`make start`
{% endnote %}

## 实验关键里程碑数据与结果

### 所安装的Qemu版本

由于qemu存在很多架构的模拟器，所以查看版本的命令并不是简单的`qemu`，而是需要输入qemu架构的全名，后面加一个`-version`（既不是`--version`也不是`-v`）。
{% asset_img qemu-version(x86-64).png %}
而输入`qemu-`，按<kbd>tab</kbd>键提示的命令竟然有这么多
{% asset_img qemu-environs.png %}
本次实验用到的是其中的一种：`qemu-system-x86_64`，可以从Makefile文件中观察到。
{% asset_img 3.png %}
修改相关文件后的运行结果。

{% gp 2-2 %} {% asset_img before-f12.png "F12按下前" %} {% asset_img after-f12.png "F12按下后" %} {% endgp %}

## 实验难点与收获

实验本身不难，但是根据需求完成内核代码修改时涉及到了不少文件，而且需要逐一确认修改完毕，而这仅仅只是输出设备相关的代码。在修改过程中，我体会到了内核代码的复杂性和系统性，对写操作系统内核的大佬们产生了由衷的敬意（确信）。

## 实验思考

本次修改过程仅涉及到了一部分文件夹中的部分文件，整个内核中还有其他的文件和文件夹，它们是干什么的呢？
