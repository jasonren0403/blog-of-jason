---
title: WinDLX 模拟器安装与使用
date: 2022-10-15 17:16:36
tags:
    - "WinDLX"
    - "计算机组成原理"
categories:
    - ["实验","计算机组成原理"]
excerpt: 建立实验环境，了解WINDLX模拟器的结构及使用
---

## 实验目的

建立实验环境，了解WINDLX模拟器的结构及使用。

## 实验原理

WinDLX软件包中带有说明文件，供安装程序时候使用。

## 实验步骤

### 安装WinDLX

由于WinDLX是一个非常古老的模拟软件，但是我们现在多用Win10和Win11系统，所以需要使用虚拟机运行该软件。Win11可以使用Hyper-V虚拟化运行方案，其界面如下所示：

{% asset_img image_qFbv2Qa_D3.png "Hyper-V管理器主界面" %}

具体启用支持的方法在这里不多说了，之后可以像VMware虚拟机一样，添加一个Windows XP的系统镜像，并等待虚拟机上操作系统安装的流程走完，即可开始使用。
{% note info %}
Hyper-V无法像VMware虚拟机那样直接将文件拖入窗口来复制，但可以先将需要用到的文件制作成vhd虚拟硬盘，然后再加载硬盘。
{% endnote %}

所给定的WinDLX文件夹如下所示，我们可以直接双击WinDLX.exe来启动软件。

{% asset_img image_sPpeg0VgHi.png %}

###  熟悉使用界面

启动软件过后，可以看到一个窗口和六个子窗口，子窗口的名称分别为Clock Cycle Diagram、Statistics、Breakpoints、Register、Code和Pipeline，双击这些子窗口或单击其上方的还原键即可看到子窗口的内容。

{% asset_img image_KxjgEY9_X1.png %}

要初始化模拟器，可以点击File 菜单中的Reset all菜单项，弹出一个“Reset DLX”对话框。然后点击窗口中的“确认”按钮即可。

{% asset_img image_YEchSWiXz5.png %}

要载入文件或代码，需要点击File菜单的Load Code or Data，在以下对话框中，依次选择Fact.s和Input.s文件，完成后单击Load，将它们加载到主存中。

{% note warning %}
注意：要先选择fact.s再选择input.s文件。程序在内存中出现的顺序很重要。

{% asset_img image_XfmJxGPgJb.png %}
{% asset_img image_jU6xYnVOwT.png "现在，单击Load按钮将它们加载到主存中" %}
{% endnote %}

成功加载文件后，会提示你重置DLX，单击“是”。这样，文件就已经装入到存储器中了。

{% asset_img image_zGYcGmsMFa.png %}

结合装载的文件，来介绍一下六大窗口的主要功能

#### Code 代码窗口

在Code窗口中，我们能够看到代表存储器内容的三栏信息，从左到右依次为：地址 (符号或数字)、命令的十六进制机器代码和汇编命令。

{% asset_img image_N5Tfs32zDs.png %}

单击WinDLX窗口的Execute（执行）按钮可以执行装入的程序，可以选择单步执行（Single Cycle，快捷键F7）或多步执行（Multi Cycles，快捷键F8）。当单步执行时，也有流水段颜色的区分（IF、ID、intEX、MEM和WB）

{% asset_img image_OAHMDNAJGN.png %}

如果需要多步执行，就在这个对话框中输入要执行的流水数目即可。

{% asset_img image_1Zzwpmaorh.png %}
#### Pipeline流水线窗口

流水线窗口主要显示了DLX处理器的内部结构，窗口下标识了DLX处理器的五个流水段和浮点操作（加/减、乘和除）的单元。

{% asset_img image_bphLLi5PTe.png %}

在运行指令时，将Pipeline窗口放大，即可看到当前与Code中所处的时刻相同的指令流水，可以清晰看到不同流水段执行的是哪条指令。

{% asset_img image_ar42V_rtMT.png %}
#### Clock Circle Diagram时钟周期图窗口

Clock Cycle Diagram窗口主要显示了运行代码时流水线的时空图，时空图反映的是不同时隙内的运行情况。

{% asset_img image_mC0USoVaqr.png %}

在非空转代码的时候，显示如下：

{% asset_img image_bYqORtx7CN.png %}

任意双击指令的一行，可以详细看到不同流水段的情况，如下图所示。

{% asset_img image_ijbidVFTvq.png %}
#### Register寄存器窗口

Register 窗口会显示各个寄存器中的内容。

{% asset_img image_oSJhNz14hV.png %}
#### Statistic统计数据窗口

Statistics 窗口提供对运行程序中任何数据的分析，主要包括模拟器中硬件配置的情况。

{% asset_img image_ph76EvKYO5.png %}

这里包含：
1.  整体指令执行情况
	{% asset_img image_RVWe28mDgc.png %}
2.  硬件配置情况
	{% asset_img image_341IwOGYJ5.png %}
3.  暂停次数、百分比和原因分析
	{% asset_img image_8KxD3jZ_MV.png %}
4.  分支次数和百分比
	{% asset_img image_JHGpyQ6kXB.png %}
5.  Load/Store指令执行情况
	{% asset_img image_4o7Y5cy8hA.png %}
6.  浮点指令执行次数和百分比
	{% asset_img image_di1vmW6RNZ.png %}
7.  trap发生次数和百分比
	{% asset_img image_A9oR1nMPWT.png %}

#### Breakpoints断点窗口

该窗口主要用来观察代码运行的情况。使用前，需要打开Breakpoints窗口（首先需要激活Breakpoints小窗口才会出现Breakpoints菜单项）来设置breakpoint，这个值代表了指令运行到流水线的哪个阶段就暂停运行。

{% gp 3-1 %}
{% asset_img image_5qA1KGdijo.png %}
{% asset_img image_sM6iUTJGxM.png %}
{% asset_img image_cgfKBRYSZU.png "从菜单中选择Set后，出现Set Breakpoint窗口" %}
{% endgp %}

如果选择了ID阶段，在Code窗口相应行会出现BID，表示程序执行到取指令IF结束译码ID开始的时候，程序将中止。

{% asset_img image_XrK7jIlqPf.png "此时，Breakpoints窗口的显示为地址、中止阶段和当前指令" %}

WinDLX可以在多种配置下工作。可以改变流水线的结构和时间要求、存储器大小和其他几个控制模拟的参数。均在Configuration菜单中完成配置。

##### Floating Point Stages配置窗口

{% asset_img image_T_5pLEA9Z3.png %}

可以设置加法、乘法和触发的单元数目和延迟。

##### Memory Size配置窗口

{% asset_img image__EPQPVWZ6k.png %}

可以设置存储空间的大小。

##### 其他配置窗口

Configuration菜单中的还有其他三种配置，它们是：Symbolic addresses、Absolute Cycle Count 和 Enable Forwarding。点击相应菜单项后，在它的旁边将显示一个小钩。
  
- 最重要的配置项为Enable Forwarding，指的是启用定向技术，定向技术是指将某个计算结果从其产生的地方直接送到其它指令需要它的地方，可以减少数据冲突引起的停顿。
