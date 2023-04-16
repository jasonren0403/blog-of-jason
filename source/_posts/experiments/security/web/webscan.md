---
title: 网络扫描 
date: 2019-11-9 20:35:17 
tags:
  - "网络空间安全"
  - "网络安全"
categories:
  - ["实验","网络安全"]

---

## 目标

了解网络扫描的原理和应用，使用xscan、nmap或nessus其中的任意一种常见的扫描软件对某主机进行端口、服务、漏洞扫描，分析数据包交互过程。
<!-- more -->
{% note danger %}
本篇文章为实验记录，仅供交流学习使用，切勿违法应用，所有文中提到的工具不提供下载。
{% endnote %}

## 原理

### 扫描

网络扫描是确认网络运行的主机的工作程序，或是为了对主机进行攻击，或是为了网络安全评估。网络扫描程序，如Ping扫射和端口扫描，返回关于哪个IP地址映射有主机连接到因特网上的并是工作的，这些主机提供什么样的服务的信息。另一种扫描方法是反向映射，返回关于哪个IP地址上没有映射出活动的主机的信息，这使攻击者能假设出可行的地址。网络漏洞扫描器是网络入侵者收集信息的重要手段，主要用于扫描目标主机识别其工作状态（开/关机）、识别目标主机端口的状态（监听/关闭）、识别目标主机系统及服务程序的类型和版本、根据已知漏洞信息，分析系统脆弱点、生成扫描结果报告。

### 主机扫描

主机扫描的目的是确定在目标网络上的主机是否可达。这是信息收集的初级阶段，其效果直接影响到后续的扫描。常见扫描手段有ICMP Echo扫描、ICMP Sweep扫描、Broadcast ICMP扫描、Non-Echo ICMP扫描等。防火墙和网络过滤设备常常导致传统的探测手段变得无效。为了突破这种限制，必须采用一些非常规的手段，利用ICMP协议提供网络间传送错误信息的手段，往往可以更有效的达到目的。

### 端口扫描

确定目标主机可达后，使用端口扫描技术，发现目标主机的开放端口，包括网络协议和各种应用监听的端口。有开放扫描、隐蔽扫描、半开放扫描三种方式。

### 操作系统识别

根据各个OS在TCP/IP协议栈实现上的不同特点，采用黑盒测试方法，通过研究其对各种探测的响应形成识别指纹，进而识别目标主机运行的操作系统。根据采集指纹信息的方式，又可以分为主动扫描和被动扫描两种方式。

### 漏洞扫描

在端口扫描后得知目标主机开启的端口以及端口上的网络服务，将这些相关信息与网络漏洞扫描系统提供的漏洞库进行匹配，查看是否有满足匹配条件的漏洞存在。通过模拟黑客的攻击手法，对目标主机系统进行攻击性的安全漏洞扫描，如测试弱势口令等。若模拟攻击成功，则表明目标主机系统存在安全漏洞。

## 步骤和关键技术

### 主机、服务和端口扫描

{% note info %}
本次使用的扫描工具为nmap6.49
{% asset_img 0.png %}
{% endnote %}

对目标机所在地址及地址段进行了Regular scan、Ping scan、Intense scan三种扫描。
{% gp 3-3 %}
{% asset_img 1.png "Regular scan" %}
{% asset_img 2.png "Ping scan" %}
{% asset_img 3.png "Intense scan" %}
{% endgp %}
另外尝试了Maimon扫描（下图）。扫描结果见“数据与分析”部分。 {% asset_img 4.png "Maimon" %}

{% note info %}
本部分3.1实验中，探测机IP为192.168.235.136，运行Windows XP系统。被探测机IP为192.168.235.138，运行Windows Server 2012系统。
{% endnote %}

### 漏洞扫描

本部分实验使用了Kali Linux自带的metasploit工具，它是一个开源的安全漏洞检测工具，可以帮助安全和IT专业人士识别安全性问题，验证漏洞的缓解措施，并管理专家驱动的安全性进行评估，提供真正的安全风险情报。
{% note info %}
本部分使用的扫描机IP为192.168.235.132，运行Kali Linux；被扫描机IP为192.168.235.136，运行Windows XP系统。
{% endnote %}

{% asset_img 6.png "metasploit主界面" %}

1. 打开metasploit主界面，输入`show exploits`查看所有漏洞的列表 {% asset_img 7.png %}

    - 按照字母顺序，列表从左到右分别是漏洞名称、披露日期、可利用性、描述，其中漏洞的命名格式是"平台/类型/具体应用简称"。由于我的Kali版本并不是最新的，所以披露日期可能比较偏早（与此同时，msf的最新版本已经是5.0了）。

2. 本次实验中，选择`mysql_start_up`漏洞进行复现。 {% asset_img 8.png %}

	{% note info %}
	#### mysql_start_up启动项提权
	在数据库中添加一个表格，将<code>vbs</code>或<code>bat</code>代码写入表中，内有<code>net user</code>创建后门用户指令，然后运行以下语句
	{% codeblock lang:sql line_number:false %}
	select \ from a into outfile "%startup%\\a.vbs";
	{% endcodeblock %} 导出脚本，并添加到系统启动项[^1]。
	{% endnote %}

3. msf下`mysql_start_up`提权只需要知道一组数据库用户名和密码，但有一定的成功几率，需要多试几次，对英文版系统支持较好。
    * 命令行下输入`use <漏洞全名>`，进入漏洞利用模式。`show targets`的结果代表了Windows上的MySQL可能会被利用。 {% asset_img 9.png %}

4. `show payloads`可以查看所有载荷列表，载荷（payloads）是一串代码中主要的操作部分，在漏洞利用中充当主要的位置。 {% asset_img 10.png "可用载荷列表" %}

	{% tabs payloads, 1 %}
	<!-- tab <code>generic/shell_reverse_tcp</code>利用方法 -->
	{% asset_img 11.png %}
	set payload "payload_name"可以设定使用的payload，show options可以查看该payload利用到的所有参数，我们所要做的是将所有required部分填上我们需要的值。
	{% grouppicture 2-2 %}
	{% asset_img 12.png %}
	{% asset_img 13.png %}
	{% endgrouppicture %}
	设置完成后，使用`exploit`命令开始利用漏洞。
	
	{% note success no-icon %}
	从输出上来看，目标机上已经被传送了一个后门文件<code>wKpyl.exe</code>，它需要我们手动清除。
	{% asset_img 14.png %} {% asset_img 15.png %}
	目标机的相应位置上确实被置入了一个这样的后门程序。
	
	* 由于漏洞披露已经有了一段时间，当我拖到宿主机准备分析时，杀毒软件报毒（这说明漏洞库已经收录了漏洞相关信息），其类型为后门病毒。这个可执行文件的部分属性如下。
		{% asset_img 16.png %} {% asset_img 17.png %}
	  * 由于这是开机启动型后门，所以需要重新启动server才会奏效。
	  {% endnote %}
	
	<!-- endtab -->
	<!-- tab <code>windows/messagebox</code>利用方法 -->
	{% note info 总结 %}
	顾名思义，就是弹个框，告诉用户“你被攻击了”。算是一个能“看得见”的有效漏洞利用。
	{% endnote %}
	和（1）类似，首先set payload，然后show options。观察参数，发现“可视化”的部分在于<code>text</code>和<code>title</code>，我们决定给它加一点特效（笑）。
	{% grouppicture 2-2 %}
	{% asset_img 18.png %}
	{% asset_img 19.png %}
	{% endgrouppicture %}
	
	{% asset_img 20.png "exploit!" %}
	输入<code>exploit</code>开始利用。观察输出，发现它也是向目标输出了一个<code>exe</code>可执行文件，放入了启动项中。
	{% asset_img 21.png %}
	
	重启Win Server后，系统弹出了这个MessageBox，证明上述漏洞被有效利用且能成功执行payload程序。 {% asset_img 22.png %}
	
	<!-- endtab -->
	{% endtabs %}

## 数据和分析

下面以Wireshark抓包的角度来分析本次实验的数据交互情况。
{% note info %}
目标主机情况如下
{% grouppicture 5-2 %}
{% asset_img true-info.png "目标机网络配置" %}
{% asset_img true-info2.png "目标机网络配置2" %}
{% asset_img true-mac.png "目标机MAC地址" %}
{% asset_img true-php.png "目标机php版本" %} 
% asset_img true-winver.png "目标机Windows版本" %}
{% endgrouppicture %}
{% endnote %}

### 基本信息扫描

{% tabs basic_scan, 1 %}
<!-- tab Regular Scan扫描 -->
从下图可以看到，这是一个典型的TCP SYN扫描，是半开放扫描。
{% asset_img 23.png %}
{% asset_img 24.png %}
扫描机不断向目标机不同端口发送SYN包，请求一次连接，而在目标机回复SYN+ACK时，自己发送一个RST包停止连接。SYN+ACK的回复代表目标机的端口正处于监听状态，可以代表端口的开启情况。

目标机的80端口和8080端口处于监听状态，是开启的。

Regular Scan的输出：

```text
Nmap scan report for 192.168.235.138
Host is up (0.0022s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
8080/tcp open  http-proxy
MAC Address: 00:0C:29:18:BE:BF (VMware)
```

{% asset_img 1-1.png %}
{% asset_img 1-2.png %}

基本上都是对的。

<!-- endtab -->
<!-- tab Ping Scan扫描 -->
我用ping scan扫描了<code>192.168.1.0/24</code>整个网段，可以看到密集的ICMP请求，是在做ping操作。
{% asset_img 28.png %}
{% asset_img 29.png %}

短时间内同时向多个IP地址发送了type13的ICMP timestamp request请求，这是一种Non-Echo的ICMP扫描。

针对每一个地址，它经历了以下过程

1. 扫描机向目标主机发送了ICMP Echo Request(type 8)数据包，等待回复的ICMP Echo Reply包(type 0)，这是ICMP Echo的扫描技术。 {% asset_img 30.png %}
2. Echo Request是同时向多个主机做出的并行扫描动作，因此，也用到了ICMP Sweep扫描。 {% asset_img 31.png %} {% asset_img 32.png %}
3. 但是，扫描结果并不太准确，192.168.235.138虽然是在线主机，但是可能是上一次扫描遗留下来的结果。我只扫描了192.168.1.0的地址段。
    {% grouppicture 2-2 %}
    {% asset_img res2-1.png %}
    {% asset_img res2-2.png %}
    {% endgrouppicture %}

<!-- endtab -->
<!-- tab Intense Scan扫描 -->

1. 先进行端口的探测，使用SYN扫描。 {% asset_img 33.png %} {% asset_img 34.png %} {% asset_img 35.png %} 与可联通的端口建立握手连接，预备后续探测。
2. 通过HTTP协议访问根目录，Server响应暴露目标机的服务类型。 {% asset_img 36.png %}
3. 通过OPTIONS请求，探求目标机的支持服务类型，紧接着目标机向其DNS服务器发动了一次向www.360se.net的解析请求。 {% asset_img 37.png %} {% asset_img 38.png %}
4. 在整个扫描期间，可以发现目标机不停地在向`133.130.101.150`这个IP发送带有SYN、ECN、CWR标志的TCP包。
5. 故意请求一个可能并不存在的地址，或构造不合法的请求方式，从返回错误中也可以暴露服务器所开设的服务信息。就算是404 Not Found，返回头中也可能携带服务信息。 {% asset_img 39.png %} {% asset_img 40.png %} {% asset_img 41.png %} {% asset_img 42.png %} {% asset_img 43.png %}
6. 使用一个已存在的网页进行验证，扫描结束。 {% asset_img 44.png %}
7. 另外，观察每次发送带有`PSH|FIN`标记的请求时，目标机回复的应答包中，确认序列号为攻击者序列数+1，可以从这里提取出目标机操作系统为Windows，这是ACK扫描——一种检测目标主机操作系统的方法。{% asset_img 45.png %}
8. 以下是扫描结果

    {% grouppicture 2-2 %}
    {% asset_img res3-1.png "端口和主机情况" %}
    {% asset_img res3-2.png "其他基本信息" %}
    {% endgrouppicture %}

{% asset_img res3-3.png %}

<!-- endtab -->
<!-- tab Maimon扫描 -->

1. 首先上扫描结果。可以看到扫描结果基本正确，操作系统也比较准确。 {% asset_img 46.png %} {% asset_img 47.png %}
2. 它比Intense扫描多扫出了一个3306端口，它是怎么发现的呢？
    {% note primary %}
    在SYN扫描时，它就发现了3306号端口（3306号端口给它发送了SYN+ACK），再一连接，Server就会回复Server Greeting，从而发现了3306号端口的存在，版本号也随Server Greeting的发出而同时泄露。
    {% asset_img 48.png %} {% asset_img 49.png %} 
    {% endnote %}

3. 预先的扫描是和（3）中一样的SYN方法。Maimon还会进行重试。
    {% grouppicture 2-2 %}
    {% asset_img 50.png "发送多个SYN包" %}
    {% asset_img 51.png "发动重试"%}
    {% endgrouppicture %}
4. 但在端口扫描前，maimon会先发送FIN+ACK请求关闭连接，这样做的目的可能是重置状态，减少干扰。 {% asset_img 53.png %}
5. 扫描期间，还可以看到目标机的TLS通信过程。含义推测是要进行443号端口相关的验证，需要用到这些协议。
    {% asset_img 52.png %} {% asset_img 55.png %} {% asset_img 54.png %}
    {% note info %} 
    这里应该是在进行操作系统指纹的探查，依据是带有PSH标志的ACK包，不过负责这里验证的IP不再是探测机的IP！
    {% endnote %} 

    做完这些检测后，被扫描机发送RST+ACK关闭连接，接下来做服务检测。验证方法与（3）类似，不再赘述。 {% asset_img 56.png %}

<!-- endtab -->
{% endtabs %}

### 漏洞扫描及利用

1. 首先利用已知的一组用户名密码登录MySQL服务器。 {% asset_img 57.png %}
2. 向表中插入二进制数据，在传之前还做了一些事情。 {% asset_img 58.png "应该是很长的sql语句的拆分" %}
3. 查询服务器的操作系统。 {% grouppicture 2-2 %} {% asset_img 59.png "询问" %} {% asset_img 60.png "响应回答" %} {% endgrouppicture %}
4. 查询操作系统临时表目录。 {% grouppicture 2-2 %} {% asset_img 61.png "询问" %} {% asset_img 62.png "响应回答" %} {% endgrouppicture %}
5. `select xxx info dumpfile` 是最关键的一步。漏洞利用关键点。由于相关防护不到位，服务器接受了这个请求，把二进制数据导出了文件，并添加到了启动项文件夹中。做完这个操作后，攻击机就发送FIN+ACK与服务器断开了链接。{% asset_img 63.png %}
    {% note info no-icon %}
    弹框时的数据交互情况类似。
    {% endnote %}

## 收获

通过使用扫描软件对于某主机进行扫描，再进行抓包分析，直观的看到了主机扫描数据交互的过程，理解了网络扫描的基本原理和不同扫描方法的特点。使用metasploit尝试了漏洞的利用，体验了漏洞的表现和侵入性。

## 补充

1. 做完“漏洞扫描”实验的metasploit部分后，我才发现metasploit是漏洞利用软件而非漏洞扫描软件。重新上网查找相关资料，发现存在扫描漏洞的nmap脚本[nmap-vulners](https://github.com/vulnersCom/nmap-vulners)，决定使用这个脚本重新完成本部分实验和分析。
2. 使用`-script`选项指定`nse`脚本位置。扫描几乎如同普通扫描，也扫出了三个开放端口。
    {% asset_img ex0.png %} {% asset_img ex1.png %} {% asset_img ex2.png %}

3. 但是没有看到漏洞相关的信息输出。我换用kali linux重新进行扫描，得到了以下结果 {% asset_img ex3.png %}

4. 以上结果表明系统确实存在了很多cve漏洞，甚至还有2019年的新漏洞，随便打开一个网页看看：http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-0211 （nmap给的页面链接过不了验证码）

	{% centerquote %}
	CVE-2019-0211是一种特权升级漏洞，在Apache HTTP Server 2.4版本2.4.17至2.4.38中，使用MPM事件，工作者或prefork，在权限较低的子进程或线程（包括进程内脚本解释器执行的脚本）中执行的代码可以执行任意代码通过操纵记分板，父进程（通常是root）的权限。非Unix系统不受影响。
	{% endcenterquote %}
	
	{% note default %}
	按说这个漏洞对于Win Server来说没影响，如果检测了目标操作系统的话，这个漏洞的重要性可能就会下降了。
	{% endnote %}

5. 对扫描过程抓包，发现前半部分与扫描端口、服务时基本一致，假如端口开放，目标机会发回SYN+ACK包，此时自己再发出RST包 {% asset_img ex4.png %}

6. 中间服务探测的过程与intense扫描基本一致，后期vulners检测工具与它自己的服务器进行了交互，可能是为了将特征与自己的数据库比对。
    {% grouppicture 2-2 %}
    {% asset_img ex5.png %}
    {% asset_img ex6.png %}
    {% endgrouppicture %}

[^1]: 来自https://www.cnblogs.com/yzloo/p/10390916.html
