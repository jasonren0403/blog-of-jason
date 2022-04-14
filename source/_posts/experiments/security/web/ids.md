---
title: 入侵检测 
date: 2019-11-25 20:12:01 
tags:
  - "网络空间安全"
  - "网络安全"
categories:
  - ["实验","网络安全"]

---

## 目的

了解入侵检测的基本原理，尝试在Windows XP下配置Snort IDS并完成基于Snort的攻击检测。尝试利用Snort规则语法自己编写简单的Snort规则。
<!--more-->
{% note danger %} 本篇文章为实验记录，仅供交流学习使用，切勿违法应用，所有文中提到的工具不提供下载。 {% endnote %}

## 原理

### 入侵检测定义

入侵检测是防火墙的合理补充，帮助系统对付网络攻击，扩展了系统管理员的安全管理能力(包括安全审计、监视、进攻识别和响应)
，提高了信息安全基础结构的完整性。它从计算机网络系统中的若干关键点收集信息，并分析这些信息，看看网络中是否有违反安全策略的行为和遭到袭击的迹象。入侵检测被认为是防火墙之后的第二道安全闸门，在不影响网络性能的情况下能对网络进行监测，从而提供对内部攻击、外部攻击和误操作的实时保护。

### Snort软件

Snort是由Sourcefire开发的一个开源的网络入侵检测系统。用C语言编写的开放源代码软件。它是一个跨平台、轻量级的网络入侵检测软件。它结合了协议、特征和基于异常的检测，是目前使用最多的IDS/IPS系统。
一条snort规则包括包含规则头和规则选项两部分；snort规则如下：

1. Snort规则应写在一行，若需多行，应用`\`来拆分。
2. 规则头包含规则的动作，协议，源和目标的IP地址和端口信息
3. 规则选项部分包含细节、具体的描述和约定。

## 实验过程

### 配置Snort环境

首先装好Snort软件。用`snort -W`命令测试安装正确性 {% asset_img 0.png 200 200 %}

{% asset_img 1.png 200 200 %} 界面上显示了“小猪嘴”和版本号，即宣告安装成功

* 运行`snort -i 1`可以进入嗅探模式，显示如下 {% asset_img 2.png 250 250 %}

* 运行`snort -de -1`进入的是数据包记录器模式。我把日志记录在了C盘的Snort/log文件夹下，此文件可以用Wireshark打开。 {% asset_img 3.png %} {% asset_img 4.png %}

* 根据实验指导书做一些配置修改后，用以下命令启动攻击检测模式。
    ```bash
    snort -c "path/to/snort.conf" -l "path/to/logdir"
    ```
    * 此时开始了攻击检测模式，可以到`C:/Snort/log`下查看告警日志。 {% asset_img 6.png 200 200 %} {% asset_img 7.png 200 200 %}
    * 不过里面没有UDP包告警的相关的信息……是因为默认的社区规则里面并没有针对这个情况进行专门配置 {% asset_img 8.png 200 200 %}

### 添加UDP规则

1. 在本地添加一条UDP规则
    ```
    alert udp any any <>$HOME_NET any (msg:”udp ids/dns-version-query”;content:”version”;)
    ```
   {% note info %}
   #### 含义
   这条规则的含义是：对一条UDP规则生成一条警报并记录这个包，此规则针对任意UDP协议的非本地任何地址任何端口双方向（HOME_NET在snort.conf中有配置，应该配置为本机的地址），发送一条信息，内容为”udp
   ids/dns-version-query”，包含”version”字符即匹配 {% endnote %}

{% asset_img 9.png %}

2. 由上图得知，我的电脑的`HOME_NET`值应该为192.168.235.136。将规则加入local.rules中，注释掉其他的rules，只运行自己写的rules。

{% asset_img 10.png %} {% note warning %}
最终规则语句调整为`alert udp any any <> $HOME_NET any (sid:xxxx;msg:"udp ids/dns-version-query";content:"version";)`
`<>`两侧必须有空格，每条规则必须有一个sid值，msg必须包含在英文引号中。 {% endnote %}

3. 在另一台虚拟机上运行UDP Flood工具，对135端口进行扫描攻击。注意所发的信息要包含`”version”`字符串，才会有报警（否则不会匹配规则）。 {% grouppicture 2-2 %} {% asset_img
   11.png %} {% asset_img 12.png %} {% endgrouppicture %}
    * alert.ids和同期所记录log用wireshark打开的结果如上图所示。

## 实验结果及分析

### 自己编写Snort规则：`mytelnet.rules`

{% note info %} 要求：记录外网对内网的Telnet连接企图，并发出下面的警告：External net attempt to access internal telnet server。 {% endnote %}

* 根据snort规则的相关编写知识，得到相应的snort规则应该为：

```
alert tcp !$HOME_NET any -> $HOME_NET 23 (msg:"External net attempt to access internal telnet server";sid:xxxx;)
```

{% asset_img 13.png 200 200 %}

* 装入local.rules进行测试，然后用外网telnet snort所在的虚拟机地址。测试结果如下：（测试机需要事先开启telnet服务）

{% grouppicture 4-4 %} {% asset_img 14.png %} {% asset_img 15.png %} {% asset_img 16.png %} {% asset_img 17.png %} {%
endgrouppicture %}

果然在`alert.ids`中有相应记录

### 利用Snort检测网络扫描攻击

为使snort具备端口扫描检测的能力，需要启用端口扫描检测插件，在snort中，这类插件通常通过预处理器的方式提供。

* 在`snort.conf`中，找到`sfportscan`预处理器，把注释去掉以开启这个预处理器。 {% asset_img 18.png %} {% note info %} 其中proto参数代表欲检测的协议，取值有tcp udp
  icmp ip all，各参数间空格分开，这里写all就行。 Memcap代表分配给端口扫描的最大字节数，数字越高，越多节点可以被跟踪。 Sense_level用于检测端口扫描的敏感度水平。取值有low medium
  high三种。low级别使用通用方法来查找响应错误，medium会检测端口扫描和过滤端口扫描（那些没有收到响应的端口扫描），high级别有最小的端口检测限制。这里设为了high。
  除此以外，还有scan_type、logfile、watchip、ignore_scanners配置项，对于本实验的需求不大。 {% endnote %}

1. 在一台攻击机上启动nmap进行扫描。同时在目标机上启动snort。 {% asset_img 19.png %} {% asset_img 20.png %} {% asset_img 21.png %} {% asset_img
   22.png %}
2. 结果只检测出了上述三个有效信息，没有像预期那样检测到扫描工具扫描的痕迹。 {% note warning %} 预期可能会检测出类似于SCAN UPnP service discover
   attempt这样的信息，可能是规则配置不力（忘记取消注释或者规则语句错误），或者是扫描行为不匹配当前本地已启用规则所致？ {% endnote %}

## 总结

通过几个小的snort配置实验，我对基于snort的入侵检测有了初步的了解，学习书写了几个简单的snort规则并测试其应用过后报警的效果。这个输出格式一般为：

```
[**] [1:sid:rev] "msg" [**]
[Classfication] [Priority:<priority>]
[time] [from[ip:port]->dest[ip:port]]
[protocol] [ttl] [tos] [id] [iplen] [dgmlen] [flags]
[各协议包额外信息]
```

## 补充

重新使用其他扫描工具完成端口扫描实验，发现可以成功检测。说明上述检测不成功的原因应该是扫描动作与预期规则不匹配。我使用了Nessus 3版本完成了这一部分的实验。 {% asset_img 24.png %}

1. 同时开启被扫描端的snort，发现除了有尝试执行shellcode以外(这个原来用nmap扫的时候也扫到了)还发出了SCAN的警告，分类信息显示为Attempted information
   leak（尝试泄露信息），基本可以确认nessus发出的扫描被我们检测出来了。 {% asset_img 25.png %} {% asset_img 26.png %}

2. 此扫描还尝试利用NETBIOS登录来取得用户权限，以获取后续漏洞信息。 {% asset_img 27.png %}
