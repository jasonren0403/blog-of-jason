---
title: DLX处理器程序设计
date: 2022-10-24 17:16:36
tags:
    - "WinDLX"
    - "计算机组成原理"
categories:
    - ["实验","计算机组成原理"]
excerpt: 学习使用DLX汇编语言编程，进一步分析相关现象
---

## 实验目的

学习使用DLX汇编语言编程，进一步分析相关现象

## 实验原理

掌握向量运算算法和编程方法。由于操作数为双精度，需要使用操作浮点双精度的指令。

本次使用到的指令为

-   ADDD浮点加指令，其命令格式为`rd,rs1,rs2`，rs1和rs2将加到rd上。
-   ADDI立即数加指令，其命令格式为`rd,rs1,immediate`，rs1和一个立即数将加到rd上。
-   SD存储浮点数指令，其命令格式为`offset(rs1),rd`，将rs1对应空间的数存储在rd上。
-   LD读取浮点数指令，其命令格式为`rd,offset(rs1)`，将rs1对应空间的数取到rd上。
-   BNEZ判断指令，其命令格式为`rs1,name`，如果CPR寄存器不为零，则继续执行name部分。

## 实验步骤

1.  自编一段汇编代码，完成两双精度浮点一维向量的加法（或乘除法）运算，并输出结果。向量长度>=16。
	```nasm
	.data
	VecLen: .word 16
	Vector1: .double 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16
	; vec1=[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16]
	Vector2: .double 2,4,6,8,10,12,14,16,18,20,22,24,26,28,30,32
	; vec2=[2,4,6,8,10,12,14,16,18,20,22,24,26,28,30,32]
	; 声明向量长度以及两个向量1、2
	Printf1: .asciiz "The result is\n"
	Printf2: .asciiz "%f\t"

	.align 2

	PrintfHead: .word Printf1
	PrintfValue: .word Printf2
	result: .space 8
	; 打印数据空间的申请

	.text

	main:
	addi r14,r0,PrintfHead
	trap 5
	lw r20,VecLen
	addi r2,r0,0

	loop: ; 循环读入向量
	ld f0,Vector1(r2)
	ld f2,Vector2(r2)
	addd f4,f0,f2
	; 主要的加法运算
	sd result,f4
	addi r14,r0,PrintfValue
	trap 5
	addi r2,r2,8
	subi r20,r20,1
	bnez r20,loop

	trap 0
	; 系统中断，输出结果 "The result is 3.000000, 6.000000, ...."
	```

    程序，运行结果为下图

	{% asset_img image_phympq_63F.png %}

	数据统计为下窗口所示：
	
	{% gp 2-2 %}
	![](image_jmdoV6noOX.png)
	![](image_nIrZXuh4OD.png)
 	{% endgp %}

2. 观察程序中出现的数据/控制/结构相关。

	{% asset_img image_30Lxy8oMOz.png %}

	数据相关：对当前指令的操作数寄存器进行操作（EX）的时候，前几条指令的运算结果还未写回（WB）结果寄存器，产生了数据相关。

	典型指令组合
	```nasm
	subi r16,r16,0x1
	bnez r16,Loop
	```
	控制相关：由于系统按照预测成功来执行指令，所以执行bnez后会马上将其下一条指令trap读取进来

	典型指令组合：
	```nasm
	bnez r20,Loop
	trap 0x0
	```
	程序中并没有出现结构相关。因为整体仅仅做了一次简单的加法。

3.  浮点运算部件带来的影响

	{% tabs floats, 1 %}
	<!-- tab 使用1个浮点加法器 -->
	{% asset_img image_CHQj5DasyE.png %}
	<!-- endtab -->
	<!-- tab 使用2个浮点加法器 -->
	{% asset_img image_mT6P512ORr.png %}
	<!-- endtab -->
	{% endtabs %}

	增加浮点加法器的数量后，程序执行的性能并未得到提升。

4.  Forward组件的影响

	{% tabs forwards, 2 %}
	<!-- tab 启用Forward组件 -->
	{% asset_img image_uFBPKTACNp.png %}
	<!-- endtab -->
	<!-- tab 禁用Forward组件 -->
	{% asset_img image_4mnDTJDUV_.png %}
	<!-- endtab -->
	{% endtabs %}

	关闭Forward后，运行时间由267个周期增加到333个周期，可计算Forward技术为该程序带来了333/267=1.25的加速比。且Forward技术明显减少了RAW stalls，降低了数据相关的数量。

5.  转移指令的影响

	{% tabs transfer, 3 %}
	<!-- tab 转移成功 -->
	{% asset_img image_OWjDOo_wJ6.png %}
	<!-- endtab -->
	<!-- tab 转移失败 -->
	{% asset_img image_vaMruizC-Z.png %}
	<!-- endtab -->
	{% endtabs %}

    成功时，已经进入取指阶段的指令会被放弃（aborted），转入转移的目标指令的取指操作，造成流水线断流；若转移失败，已经进入取指阶段的指令继续进入译码阶段，流水线不断流。

