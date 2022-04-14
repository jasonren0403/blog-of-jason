---
title: 虚拟专用网实验 
date: 2019-12-05 20:52:04 
tags:
  - "网络空间安全"
  - "网络安全"
categories:
  - ["实验","网络安全"]

---

## 目标

了解什么是虚拟专用网（VPN），知道VPN的原理和应用。通过实验理解IPSec和VPN的关系和必要性。
<!-- more -->
{% note danger %} 本篇文章为实验记录，仅供交流学习使用，切勿违法应用，所有文中提到的工具不提供下载。 {% endnote %}

## 原理

### 虚拟专用网(VPN)[^1]

虚拟专用网（Virtual Private
Network，VPN）是利用Internet等公共网络的基础设施，通过隧道技术，为用户提供的与专用网络具有相同通信功能的安全数据通道。“虚拟”是指用户无需建立各自专用的物理线路，而利用Internet等公共网络资源和设备建立一条逻辑上的专用数据通道，并实现与专用数据通道相同的通信功能。“专用网络”是指虚拟出来的网络并非任何连接在公共网络上的用户都能使用，只有经过授权的用户才可使用。

该通道内传输数据经过加密和认证，可保证传输内容的完整性和机密性。

### IPSec

互联网安全协议（英语：Internet Protocol
Security，缩写为IPsec），是一个协议包，通过对IP协议的分组进行加密和认证来保护IP协议的网络传输协议族（一些相互关联的协议的集合）。由认证头（AH）、封装安全载荷（ESP）、安全关联（SA）组成。

IPSec被设计用来提供入口之间和端到端之间的通信安全，无论哪一种模式都可以用来构建虚拟专用网（VPN）。

## 步骤和关键技术

### IPSec协议配置

{% note info %} Windows XP端的IP为192.168.235.136，Windows 2000端的IP为192.168.235.141 {% endnote %}

1. 在WindowsXP下打开“控制面板→管理工具→本地安全策略”，找到“IP安全策略，在本地计算机”。双击进入。然后右击“安全服务器”，点击指派规则。 {% asset_img 1.png %} {% asset_img 2.png
   %}
2. 双击“安全服务器”进入其设置，双击“所有ICMP通讯量”进入规则设置。 {% asset_img 3.png %}
3. 在“筛选器操作”选项卡中，选择“需要安全”。 {% asset_img 4.png %}
4. 在“身份验证方法”选项卡中，添加一个共享密钥。记住此密钥，在另一端要做同样的设置。 {% asset_img 5.png "身份验证方法选项卡" %} {% tabs key, 1 %}

<!-- tab XP端 -->
{% asset_img 6.png "设置共享密钥" %}
<!-- endtab -->
<!-- tab 2000端 -->
{% asset_img 7.png "设置共享密钥" %}
<!-- endtab -->
{% endtabs %}

5. 只在一端配置时，ping端会直接显示Request timed out。双端配置时，会显示Negotiating IP Security。正确配置时，会先显示Negotiating IP
   Security，然后才会显示ping的reply，代表双端成功建立了连接。可以在win2000端的IP安全监视器中看到连过来的xp设备。 {% grouppicture 3-3 %} {% asset_img 8.png "
   只配置了一边" %} {% asset_img 9.png "双端配置" %} {% asset_img 10.png "完全正确的配置" %} {% endgrouppicture %} {% asset_img 11.png "
   查看链接的Windows XP设备" %} {% note info %} 当然，也可以配置专用的IPSec安全策略，不使用系统默认值。过程类似，不再赘述。 {% endnote %}

### OpenVPN配置

{% note info %} 本部分实验使用WindowsXP和Windows2000共同完成，WindowsXP充当服务端，Windows2000充当客户端。 {% endnote %} {% note warning %}
事先下载安装好openssl库，否则会报“命令不存在”的错误。 {% endnote %}

1. 首先安装OpenVPN。安装过程一路下一步即可。
2. 将安装目录easy-rsa下`vars.bat.sample`文件换成自己的信息，然后更名为`vars.bat`。

```shell vars.bat
...
set <KEY>=<VALUE>
```

{% asset_img 12.png %}

3. 将`openssl.cnf.sample`更名为`openssl.cnf`。 {% asset_img 13.png %}
4. 在easy-rsa文件夹下，先后运行`vars`和`clean-all.bat`命令，以进行初始化。 {% asset_img 14.png %}
5. 生成CA根证书。 {% asset_img 15.png %}
6. 生成密钥交换协议算法参数。 {% asset_img 16.png %}
7. 生成服务端和客户端证书并签发十年有效期。 {% tabs certificate, 1 %}

<!-- tab server -->
{% codeblock lang:shell line_number:false %} build-server-key.bat server {% endcodeblock %} {% asset_img 17-1.png %} {%
asset_img 17-2.png %}
<!-- endtab -->
<!-- tab client -->
{% codeblock lang:shell line_number:false %} build-key.bat client {% endcodeblock %} {% asset_img 18-1.png %} {%
asset_img 18-2.png %}
<!-- endtab -->
{% endtabs %}

8. 生成 ta key {% codeblock lang:shell line_number:false %} openvpn --genkey --secret keys/ta.key {% endcodeblock %} {%
   asset_img 19.png %}
9. 以上步骤完成后，keys中包含了这么多文件 {% asset_img 20.png %}
10. 配置`server.ovpn`文件，将它们连同服务器证书一起复制到config文件夹下。 {% asset_img 21.png %}
11. 客户端配置client.ovpn文件。然后将客户端证书复制到config目录下。 {% asset_img 22.png %}
12. 然后就可以右击任务栏的图标，点击Connect进行连接了。 {% grouppicture 2-2 %} {% asset_img 23.png %} {% asset_img 24.png %} {%
    endgrouppicture %}

## 数据和分析

### IPSec相关数据包分析

1. 在只配置了其中一端的IPSec时，服务器发回的包都是Identity Protection（Main Mode）的ISAKMP包，其中每一段payload都有一个next payload字段，指定下一个载荷包的内容。在Vendor
   ID中，存在一个`MS NT5 ISAKMPOAKLEY`字段，这里是目标操作系统的信息（所以可能有泄露os的利用！）。 {% grouppicture 2-2 %} {% asset_img 25-1.png "Exchange
   Type" %} {% asset_img 25-2.png "Vendor ID 可暴露" %} {% endgrouppicture %}
2. 在SA段中，信息被以IPSec的方式来解析，Identity Only被设为`True`，Payload中的Transform段是协商的密钥信息。 {% asset_img 25-3.png "SA段" %} {% asset_img
   25-4.png "Proposal部分" %} {% asset_img 25-5.png "协商参数" %}
3. 双方交互还用到了informational包，包的信息如下。 {% asset_img 25-6.png "Informational包" %}
4. 在双方都配置了IPsec的情况下，连接抓包情况如下。 {% asset_img 26-1.png %}
5. 一开始是quick mode，包的信息如下。这时信息已经是加密的了。 {% asset_img 26-2.png %}
6. 后面双方的数据都以ESP包的形式来传送了，由于双方有了约定，后面的传输都使用相同的SPI。 {% grouppicture 2-2 %} {% asset_img 26-3.png %} {% asset_img 26-4.png
   %} {% endgrouppicture %}

### OpenVPN连接过程日志分析

{% asset_img 27.png "建立连接" %}

1. 在建立连接时，双方使用了UDP连接（我服务器端配置的是udp服务器），可以看出其中的数据是加密的。
2. 观察生成的log文件，发现client端使用了1040以上的端口与服务器端建立了连接，在连接时，双方走了个人信息的认证。还商讨了有关密码算法的信息，比对了options hash。 {% grouppicture 2-2 %} {%
   asset_img 28.png %} {% asset_img 29.png %} {% endgrouppicture %}

## 收获

通过配置虚拟专用网和抓包分析，我理解了VPN相对于普通连接的安全性，明白了IPSec在其中扮演的重要角色。

但是，其中的IKE密钥协商过程还是有一些安全缺陷的。由于响应方需要时间计算模幂参数，占用大量cpu和内存空间，这是一个瓶颈，很容易受到拒绝服务攻击。主模式下无法提供身份隐藏保护，攻击方可以伪造响应方地址，并与发起方协商D-H公开值。

## Packet-tracer实验重要截图

{% asset_img 30.png "最终网络图" %}

* 注意交换机需要连接到PC的快速以太网口上（需要关机→修改物理设置（添加端口）→开机） {% asset_img 31.png %}
* IPSec相关设置（router0端，另一端当然是相同的，除了address为10.0.0.1） {% asset_img 32.png %}
* Ping连接效果，可以抓到ISAKMP类型的数据包，其中数据为加密状态。 {% asset_img 33.png %} {% grouppicture 2-2 %} {% asset_img 20.png %} {% asset_img
  21.png %} {% endgrouppicture %}

[^1]: https://blog.csdn.net/weixin_41924879/article/details/101365257
