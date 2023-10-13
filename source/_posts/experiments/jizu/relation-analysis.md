---
title: 指令流水线相关性分析
date: 2022-10-15 17:16:36
tags:
    - "WinDLX"
    - "计算机组成原理"
categories:
    - ["实验","计算机组成原理"]
excerpt: 通过使用WINDLX模拟器，对程序中的三种相关原理进行观察，并对使用专用通路，增加运算部件等技术对性能的影响进行考察，加深对流水线和RISC处理器的特点的理解
---

## 实验目的

通过使用WINDLX模拟器，对程序中的三种相关原理进行观察，并对使用专用通路，增加运算部件等技术对性能的影响进行考察，加深对流水线和RISC处理器的特点的理解。

## 实验原理

指令流水线中主要有结构相关、数据相关、控制相关。相关影响流水线性能。

### 数据相关

由于相邻的两条或多条指令使用了相同的数据地址而发生的关联。

### 结构相关

结构相关是指因为程序的执行方向可能被改变而引起的相关。可能改变程序执行方向的指令通常有无条件转移、一般条件转移、复合条件转移、子程序调用、中断等。

### 控制相关

控制相关是指由转移指令引发的相关。

## 实验步骤

{% note info %}
#### 实验准备
将浮点运算的延时设为4个周期
{% asset_img image_aXqazps9yb.png %}
{% endnote %}

选择File→Load Code or Data，将input.s文件和fact.s文件按下图顺序装入主存

{% asset_img image_6o0VQk_-Ji.png %}

使用WinDLX模拟器，对Fact.s做如下分析：

### 观察程序中出现的数据/控制/结构相关，指出程序中出现上述现象的指令组合
#### 数据相关及其组合

5个周期里，Clock Cycle Diagram窗口的时空图和Pipeline窗口中的流图第一次出现了橘黄色的R-Stall
	
{% asset_img image_nALPeNEB9z.png %}
	
点击 `seqi r5,r3,0xa`，出现的具体信息如下图
	
{% asset_img image_l4MCfQqj2b.png %}
	
`lbu r3,0x0(r2)` 要在WB周期内写回R3的数据，而下一条指令`seqi r5,r3,0xa`要在intEX周期内读取r3的数据，发生了写后读RAW，所以为了避免冲突，`seqi r5,r3,0xa` 的 `intEX` 指令延迟了一个周期进行。

指令组合为
```nasm
lbu r3,0x0(r2)
seqi r5,r3,0xa
```

#### 控制相关及其组合

在第4个时钟周期，第一条命令处于MEM段，第二条命令处于intEX段，第四条命令处于IF段，而第三条命令显示为aborted。

{% asset_img image_QGI1hfUiiE.png %}

发生aborted的原因为，`jal InputUnsigned`是无条件分支指令，但只有在第三个时钟周期，`jal`指令被译码后才知道。这时，下一条指令`movei2fp`已经取出，但需执行的下一条指令在另一个地址处，因此，`movi2fp`的执行应该被取消，在流水线中留下气泡，此处为控制相关。

指令组合为
```nasm
addi r1,r0,0x1000
jal InputUnsigned
movi2fp,f10,r1
sw SaveR2(r0),r2
```

#### 结构相关及其组合

从Clock Cycle Diagram中，可以看出指令stall了4个周期

{% asset_img image_JX5gYYzGzj.png %}

`addi r2,r2,0x1`与前面的一条指令`add r1,r1,r3`发生了结构相关。由于上一条指令数据相关需要停4个周期，在ID段停滞，不能进入intEX段，故 `addi r2,r2,0x1`也不能进入ID段，译码部分被占用，故发生了结构相关。

{% asset_img image_Zwun2HbC43.png %}

相关指令组合
```nasm
add r1,r1,r3
addi r2,r2,0x1
```

### 考察增加浮点运算部件对性能的影响

{% tabs results,1 %}
<!-- tab count=1 -->
浮点运算部件的Count均为1

{% gp 2-2 %}
![](image_q0ciOGmpYn.png)
![](image_b9tWK4nJeR.png)
{% endgp %}
<!-- endtab -->
<!-- tab count=2 -->
浮点运算部件的Count均为2

{% gp 2-2 %}
![](image_9uzcYEaJhk.png)
![](image_jD1hxscOAo.png)
{% endgp %}
<!-- endtab -->
{% endtabs %}

比较各个数据，发现没有变化，无论怎么增加统计结果都一样，由此可见，在该程序中，浮点运算部件的增减对效率没有影响。原因在于，此程序浮点计算指令没有重叠，所以并行度没有增加，性能没有提高。
### 考察增加forward部件对性能的影响

{% tabs results-2, 2 %}
<!-- tab 不启用forward组件 -->

![](image_qREd2veXyT.png)
<!-- endtab -->
<!-- tab 使用forward组件 -->

{% gp 2-2 %}
![](image_aJF5rKvDFD.png)
![](image_5MH0T4ijzq.png)
{% endgp %}
<!-- endtab -->
{% endtabs %}

由以上截图可以发现：
-   增加forward组件后所执行总的时钟周期相应有所减少
-   增加forward组件使得Stall的总数有所下降，RAW的总时钟周期从29.09%下降到了13.64%
	
总之，使用forward部件后，数据相关减少，性能得到一定的改善

###  观察转移指令在转移成功和转移不成功时候的流水线开销

floating设置为2，4，forward启动时条件转移结果如下图：

![](image_IBeOkazciZ.png)

转移指令一共6条，成功转移2条，不成功转移4条。

若转移不成功，对流水线执行无影响，流水线吞吐率和效率没有降低。若转移成功，则要废弃预先读入的指令，重新从转移成功处读取指令，每执行一条条件转移指令，一条x段流水线就有x-2个流水线被浪费掉，执行效率降低，性能有一定损失。

## 实验总结

流水线技术可以提高计算机执行效率，但也有很多不足，当发生一些相关的时候执行就需要延迟，这会影响到流水线的吞吐率，而要保证高效，就需要对代码进行优化处理，这是需要个人的努力的。
