---
title: ARP欺骗 date: 2019-09-29 0:06:00 tags:

- "网络空间安全"
- "网络安全"
  categories:
- ["实验","网络安全"]
- ["漏洞","协议漏洞"]

# password: ARP
---

## 目标

尝试一次本地ARP欺骗攻击，掌握ARP协议及其利用。学会使用ARP攻击工具并进行分析。
<!--more-->
{% note danger %} 本篇文章为实验记录，仅供交流学习使用，切勿违法应用，所有文中提到的工具不提供下载。 {% endnote %}

## 原理

### ARP协议

在网络中，每台主机会有一个MAC地址，而每个网络位置会有一个专属于它的IP地址。ARP协议是将IP地址解析为MAC（硬件地址）的协议。IP地址和MAC地址不存在任何简单的映射关系，网络中的主机也在不停变化中，映射关系在不断改变，ARP协议利用了主机的高速缓冲，维护了一个从IP地址到硬件地址的映射表，这个映射表在动态更新。
若主机A要与主机B通信，首先会从主机A的缓冲中寻找B的IP地址，如果有，就查出B对应的MAC地址。若无，就在局域网中广播ARP请求（其中有B的地址），所有主机都会收到该请求。B主机收到以后，向A主机回复IP地址。B主机也会记录下A的IP地址和MAC地址，以便之后双方高速通信。

### ARP欺骗攻击

ARP高速缓存根据所接收到的ARP协议包随时进行动态更新，它是无状态的协议，不会检查自己是否发过请求包，只要收到目标MAC是自己的ARP响应数据包或ARP广播包，都会接受并缓存。
ARP协议没有认证机制，只要接收到的协议包是有效的，主机就无条件地根据协议包的内容刷新本机ARP缓存，并不检查该协议包的合法性。 因此攻击者可以随时发送虚假ARP包更新被攻击主机上的ARP 缓存，进行地址欺骗或拒绝服务攻击。

### ARP欺骗攻击的类型

* 拒绝服务攻击：构造响应包，指定协议地址为关键服务的IP地址，而MAC地址为不存在的虚拟地址。这样，局域网转发给关键服务的数据包都会丢失（因为对应物理地址不存在）。
* 广播攻击：构造响应包，其中协议地址为网关IP，而MAC地址为广播地址。如此，所有发往网关的数据包都将被广播出来，可能会占用过大带宽，干扰服务，也会泄露局域网结构。
* MAC洪泛攻击：攻击者与目标机连接在同一个集线器时，可以将网卡设置为混杂模式，从而能够接收所有经过它的数据流，而不论其目的地址是否是它。当其在交换机不同的端口时，攻击者可向交换机发送大量不同源
  MAC地址的数据包，使得交换机内存不足以存放正确的MAC地址和物理端口号的映射关系。交换机降级为集线器模式，从而可以继续使用广播攻击嗅探网络结构。
*
中间人攻击：攻击者向目标发送虚假应答包，告诉主机A“主机B的MAC地址是MacC（攻击者MAC）”，告诉主机B“主机A的MAC地址是MacC（攻击者MAC）”，此时A和B间通信的数据均被C嗅探，而A和B不能得知这一点（C会将数据正确地重定向至正确位置）。

## 实验过程

### 使用ARP攻击工具攻击

{% note info %} 本实验使用WinArpAttacker3.7.0版完成。监测机的两块网卡分别为： 192.168.235.1 00-50-56-C0-00-08 192.168.17.1 00-50-56-C0-00-01
操作系统为Windows 10 64Bit。 {% endnote %}

#### 禁止上网

{% note warning %} 目标机(A)地址为192.168.235.133 00-0C-29-E7-58-4D 操作系统windows2000。 {% endnote %}

{% asset_img 4baninternet.jpg %} 在WinArpAttacker中，扫描到目标主机后，选择攻击->禁止上网，发现Attack处变为”BanGateway”，说明攻击过程已经开始。目标机的上网受到了一定的影响。
{% asset_img 4bannedinternet2.jpg %}

#### IP冲突

{% note warning %} 目标机(A)地址为192.168.235.133 00-0C-29-E7-58-4D 操作系统windows2000。 {% endnote %} 扫描到需要攻击的主机后，选择攻击->
定时IP冲突（不断IP冲突貌似也可以，只是一次发包量的不同，定时IP冲突可以方便看到过程），Attack处变为IPConflict，证明攻击过程开始。 {% asset_img 5ipconflict.jpg %}

#### 中间人攻击

{% note warning %} 目标机(A)地址为192.168.235.133 00-0C-29-E7-58-4D 操作系统windows2000。 另外一个被监测机(B)地址为192.168.235.134
00-0C-29-9D-2D-F7 操作系统为Red Hat Linux Enterprise 7.2 64Bit。 {% endnote %}
顾名思义，中间人攻击需要有两台主机在互相通信（实验中用的是ping）。所以布置好另外一台主机，先行测试互ping成功后，开启WinArpAttacker，选择两台要监听的主机，选择攻击->
监听主机通讯。Attack处会变为SniffHosts。 {% asset_img 6sniffering.jpg %} 然后让两个虚拟机之间互相ping。与此同时打开wireshark准备进行抓包分析。 {% asset_img
6snifferatob.jpg %} {% asset_img 6snifferbtoa.jpg %}

### 编程进行ARP攻击

本机安装winpcap及其开发包环境，编程运行截图如下： {% asset_img 7pcap1.jpg %}

{% asset_img 7pcap2.jpg 200 200 %}

## 实验结果及分析

### 查看本机(A)的ARP缓存表

{% asset_img 2beforesend.jpg %} 第一行的IP地址恰好是监测机的IP之一，MAC地址也是监测机的硬件地址。类型为动态。此检测结果也表明其与监测机在同一网段下(192.168.235.*)。

### 禁止上网

{% asset_img 4bannedinternet1.jpg %} 当执行禁止上网攻击后，靶机的arp记录中多了一条`0.0.0.0->01-01-01-01-01-01`
的动态路由。经查阅资料可得知，在这里，0.0.0.0表示“本网络的本主机”，也表示默认的路由，即所有不满足路由表其他IP要求的路由均由该地址进行处理。本主机的MAC地址被指定了一个非法的值（01-01-01-01-01-01），理论上会导致发往本机的所有包均丢失。
经观察抓包记录，发现监测机不断在向外部广播请求`192.168.235.2`的地址，然而一直得不到回复。 {% asset_img 4bannedinternet3.jpg %} {% asset_img
4bannedinternet4.jpg %} {% note warning %}
但是，本次测试的这个攻击的强度并没有想象中的大，在虚拟机中也不是所有的网都上不了，可以访问百度主页，只是访问不了贴吧子域名。可能是这部分的缓存完好无损，恰好不需要0.0.0.0来默认路由的原因。 {% endnote %}

### IP冲突

{% asset_img 5ipconflict3.jpg %} 查看arp列表并没有异常的地方。
检查attacker软件的日志，发现了在IP冲突攻击前，目的机的MAC地址被改成了01-01-01-01-01-01。本地由于开了保护所以又将MAC地址改为了目的机的源MAC地址。 {% asset_img 5ipconflict4.jpg
%} 检查抓包记录，发现目标机发送了一次GARP请求，从192.168.235.133发到了192.168.235.133，这个请求通常用于确认有没有其他主机的IP与它相同。 {% asset_img 5ipconflict5.jpg %}
{% note danger %} 感觉不太对，为什么这个截图和禁止上网的那么像，有GARP请求但没有IP Conflict错误……是不是抓包时用错网卡了？ {% endnote %}

目的机在IP冲突开始的一瞬间报了一个错误。除此以外没有别的影响。 {% asset_img 5ipconflict2.jpg %}

### 中间人攻击

在靶机开心的互ping时，看到attacker软件上的日志有了这样两个事件。其中一个是修改目的IP，监测机192.168.235.1在两个主机192.168.235.133和192.168.235.134间担任了转发者的角色，两个主机并不知道它们通信时192.168.235.1在做它们的“网关”。第二个是修改MAC映射，把两个主机的对方IP对应的MAC地址改成监测机自己的MAC网关，这样就实现了对两个机子通信的监听。
{% asset_img 6sniffering2.jpg %} {% asset_img 6sniffering3.jpg %}
对两个机子的抓包结果看不到任何监听的痕迹。从两个目标机的角度来看，它们发现不了被监听的痕迹，也没有相关的第三方流量。 {% asset_img 6sniffer.jpg %} {% asset_img 6sniffera.jpg %}

### Winpcap 编程攻击

{% asset_img 7pcap4.jpg %} 攻击效果如上图，可以看到，成功地实现了MAC和IP地址的碰撞，它们全部碰撞在同一网关地址00-50-56-c0-00-08上，造成了冲突。
编程的关键在于ARP包的构建，由于winpcap提供了发包的api `int pcap_sendpacket(pcap_t *, const u_char *, int)`，我们剩下的工作就是构建正确的协议头。

* 根据ARP协议包的构成，我们有

```c arp_header.h
pkt->eth_hdr.type = htons(0x0806);
pkt->arp_hdr.hardware_type = htons(0x1);
pkt->arp_hdr.protocol_type = htons(0x800);
pkt->arp_hdr.hardware_len = 6;
pkt->arp_hdr.protocol_len = 4;
pkt->arp_hdr.option = htons(0x2);
```

`0x0806`代表该帧封装了arp包，硬件类型为以太网`0x1`，协议类型为IP`0x0800`，硬件地址为MAC地址，其长度为6，协议地址为IP地址，其长度为4，此包封装了ARP应答，值为`0x2`。
然后，在这里，构造packet的目标IP和MAC地址。

```c arp.c
if (flag == TO_VICTIM_HOST) {
    for (i = 0; i < MAC_LEN; i++)
        pkt->eth_hdr.dst_mac[i] = pkt->arp_hdr.dst_mac[i] =  VICTIM_MAC[i];
    pkt->arp_hdr.src_ip = inet_addr(GATEWAY_IP);
    pkt->arp_hdr.dst_ip = inet_addr(VICTIM_IP);
}
else if (flag == TO_VICTIM_GATEWAY) {
    for (i = 0; i < MAC_LEN; i++)
        pkt->eth_hdr.dst_mac[i] = pkt->arp_hdr.dst_mac[i] = GATEWAY_MAC[i];
    pkt->arp_hdr.src_ip = inet_addr(VICTIM_IP);
    pkt->arp_hdr.dst_ip = inet_addr(GATEWAY_IP);
}
```

最后在一个循环中，不断发出arp请求包。

```c arp.c
while (1) {
    printf("Press ctrl+C to stop attack...\n");
    if(attack_host){
        if (pcap_sendpacket(handle, (unsigned char *)&pkt_host, sizeof(ARP_PKT)) != 0)
            fprintf(stderr,"\nError sending the packet: \n", pcap_geterr(handle));
        else
            printf("attack %s...\n", VICTIM_IP);
    }
    if(attack_gateway){
        if (pcap_sendpacket(handle, (unsigned char *)&pkt_gateway, sizeof(ARP_PKT)) != 0)
            fprintf(stderr,"\nError sending the packet: \n", pcap_geterr(handle));
        else
            printf("attack %s...\n", GATEWAY_IP);
    }
}
```

前述截图中输出了attack xxx，说明ARP包被成功发送。

## 总结

### 软件使用方面

1. WinArpAttacker的兼容性在win10中略差，启动前需要先行调整为兼容模式，以防止闪退。
2. WinArpAttacker扫描主机时，要注意选择虚拟机所在的VMware网卡，否则扫描不到虚拟机，同时要注意仔细核对目标ip和mac地址，防止其对正常影响的监测机产生影响，误伤到自己或其他无辜的局域网用户。

### ARP知识方面

掌握了ARP网络包的组成和构建，并尝试在结构体的帮助下，使用C语言编程实现ARP请求包的发送。
