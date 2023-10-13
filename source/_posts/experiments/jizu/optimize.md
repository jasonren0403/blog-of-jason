---
title: 代码优化
date: 2022-10-24 19:49:56
tags:
    - "WinDLX"
    - "计算机组成原理"
categories:
    - ["实验","计算机组成原理"]
excerpt: 学习简单编译优化方法，观察采用编译优化方法所带来的性能的提高
---

## 实验目的

学习简单编译优化方法，观察采用编译优化方法所带来的性能的提高。

## 实验原理

采用静态调度方法重排指令序列，减少相关，优化程序。

## 实验步骤

### 用静态调度算法手工优化代码

选择上一个实验的两双精度浮点一维向量的加法运算代码作为优化对象，优化后的代码如下所示：	
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
addi r2,r0,0
lw r20,VecLen
;addi r2,r0,0

loop: ; 循环读入向量
ld f0,Vector1(r2)
ld f2,Vector2(r2)
; 插入指令，避免f2的RAW
addi r2,r2,8
subi r20,r20,1

addd f4,f0,f2
; 主要的加法运算
sd result,f4
addi r14,r0,PrintfValue
trap 5
;addi r2,r2,8
;subi r20,r20,1 提前到了前面
bnez r20,loop

trap 0
; 系统中断，输出结果 "The result is 3.000000, 6.000000, ...."
```

### 分析已优化程序中出现的数据/控制/结构相关

{% tabs res1 %}
<!-- tab 优化前 -->
![优化前程序](image_czojEXSniG.png "优化前程序")
<!-- endtab -->
<!-- tab 优化后 -->
![优化后程序](image__c0vohBdpb.png "优化后程序")
<!-- endtab -->
{% endtabs %}

优化后的程序的时钟周期从267个减少到了235个，RAW相关也从48个大大减少到了16个，控制相关和结构相关的数量没有改变，由此可见本次所作代码优化对数据相关的改善最大。 

### 考虑增加浮点运算部件对于性能的影响

{% tabs res2 %}
<!-- tab 1个浮点部件 -->
![1个浮点部件](image_Da5MgPENTq.png "1个浮点部件")

![1个浮点部件（图2）](image_B7x4cW0ajI.png "1个浮点部件（图2）")
<!-- endtab -->
<!-- tab 4个浮点部件 -->
![4个浮点部件](image_G26kibZeDp.png "4个浮点部件")

![4个浮点部件（图2）](image_HpO1C90ynd.png "4个浮点部件（图2）")
<!-- endtab -->
{% endtabs %}

同一段代码执行相同步，但是经过对比发现浮点运算部件的增减对于程序执行效率没有影响，可能是因为程序中不存在结构相关，所以并行度没有增加，系统性能没有提升。

### 考虑增加forward部件对性能的影响

{% tabs res3 %}
<!-- tab 使用Forward组件 -->
![使用Forward组件](image_ksxwWa7qjZ.png "使用Forward组件")
<!-- endtab -->
<!-- tab 不使用Forward组件 -->
![禁用Forward组件](image_9ROBHMR9xL.png "禁用Forward组件")
<!-- endtab -->
{% endtabs %}

使用forward组件后，执行的cycles比不适用forward组件要少，且产生的RAW stalls有明显减少，程序执行的效率提高了很多。在代码优化前，RAW相关占48个stalls，在优化后，RAW现象大量减少，数据相关明显改善，提高了代码执行效率。

### 观察转移指令在转移成功和转移不成功时的流水线开销

![Conditional Branches 输出](image_55nB__dTOL.png "Conditional Branches 输出")

本次实验中，转移成功的几率比较大。由于系统按照预测成功来执行指令，当判断转移不成功时，系统对trap指令进行的操作全部作废，转而去执行别的指令。这里代码优化对于转移指令没有影响。
