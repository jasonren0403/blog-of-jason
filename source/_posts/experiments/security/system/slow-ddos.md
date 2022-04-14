---
title: 慢速DDoS攻击实验 
date: 2019-11-3 0:05:40 
tags:
  - "网络空间安全"
  - "信息系统安全"
categories:
  - ["实验","信息系统"]
  - ["漏洞","DDoS攻击"]

---

## 实验目的

了解tomcat-jsp网站架构，学习慢速HTTP POST攻击原理和防护。
<!-- more -->

## 实验内容

1. 使用tomcat8.0和mysql5.7搭建java语言开发的网站。
2. 用Tor's hammer 1.0工具对第一步搭建的网站发动慢速HTTP POST攻击。

## 实验环境

{% asset_img env.jpg %}

## 实验要求

完成（二）实验内容并回答课后问题。

## 实验步骤和结果

### 网站搭建

1. Tomcat需要java环境运行，所以需要先安装java运行环境jre，安装过后环境显示如下 {% asset_img pre0.png %}
2. 安装Tomcat工具，Tomcat 服务器是一个免费的开放源代码的Web 应用服务器，属于轻量级应用服务器，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调试JSP 程序的首选。 {% asset_img
   pre1.png "留意到Tomcat使用的http默认端口为8080，而不是80。这意味着我们如果一会要测试安装效果的话，要输入本机IP:8080来查看" %}
3. 安装过后，在Properties中的General选项卡中，点击Start开始服务 {% asset_img pre2.png %}
4. 访问`localhost:8080`，测试安装情况 {% asset_img pre3.png %}
5. 把目标网站的`ROOT`目录拷贝到tomcat安装目录下的`webapps`目录中并覆盖。 {% note primary %} tomcat启动时会加载webapps目录下的应用程序，其中ROOT目录是网站的根目录，即<code>
   /</code>
   {% endnote %} {% asset_img pre4.png %}
6. 准备数据，将`fenxiao.sql`导入到mysql数据库中。 {% note info %} 在mysql5.7命令行中应该使用以下代码进行导入操作 {% codeblock lang:sql line_number:false
   %} create database fenxiao; use fenxiao; source fenxiao.sql; {% endcodeblock %} {% endnote %} {% grouppicture 2-2 %}
   {% asset_img pre5.png %} {% asset_img pre6.png %} {% endgrouppicture %}
7. 配置`database.properties`文件
    * 指定`jdbc.user`和`jdbc.password`为相应mysql数据库的配置，`jdbcUrl`的本机地址部分要填写为`localhost`而非本机IP（因为`root@localhost`代表了`root`
      只允许本机登录，若要从任何地方登录需要把`localhost`改为通配符`%`）。 {% asset_img pre7.png %}
    * 对于`mysql 6+`，`driver`需要填写为`com.mysql.cj.jdbc.Driver`。

8. 测试访问网站 {% asset_img pre8.png "网站访问成功" %}

### Tor's Hammer 工具攻击

1. 打开攻击机，启动其中的torshammer工具。 {% asset_img attack0.png %}
2. 使用命令`python torshammer.py -t 192.168.235.138 -r 201 -p 8080`发动攻击。 {% asset_img attack1.png %}
3. 目标机的cpu直接飚上100%，Tomcat的占用率直接提升至几乎100%，证明此种攻击确实可以短时间内侵占服务器大量资源。 {% grouppicture 3-1 %} {% asset_img attack2.png %} {%
   asset_img attack3.png %} {% asset_img attack4.png %} {% endgrouppicture %}
4. 网页也卡死在加载中了。 {% asset_img attack5.png %}
5. 断开攻击后，目标机的cpu立刻恢复原状，网页也可以打开了。 {% asset_img attack6.png %}

## 附录（可选）

### 问题回答

1. 了解原理后，应该通过什么方式来防范慢速DDOS攻击呢？
    - 前述工具是靠向目标网址POST一个请求，然后将请求拆分为一些极少量字符的payload，且保持连接来占用目标服务器的资源。 {% asset_img q1-1.png %}
    - 这些请求具有很长的`Content-length`，但每次只发一个字符，还被设定为`keep-connection`，注定会占用很多资源。
    - 前面的攻击参数中，`-r`被设定为201，是因为tomcat默认只支持同时支持200个连接，那么，把这个连接参数调大，一定程度上就可以缓解攻击，但这个方法并不是绝对的。
    - 在tomcat配置文件`server.xml`中的`<Connector ... />`配置中，和连接数相关的参数有：
        * `minProcessors`：最小空闲连接线程数，用于提高系统处理性能，默认值为10
        * `maxProcessors`：最大连接线程数，即：并发处理的最大请求数，默认值为75
        * `acceptCount`：允许的最大连接数，应大于等于`maxProcessors`，默认值为100
        * `enableLookups`：是否反查域名，取值为：`true`或`false`。为了提高处理能力，应设置为`false`
        * `connectionTimeout`：网络连接超时，单位：毫秒。设置为`0`表示永不超时，这样设置有隐患的。通常可设置为`30000`毫秒。
    - 原来tomcat最大连接数取决于`maxConnections + acceptCount`的值。如果要加大并发连接数，应同时加大这两个参数。
    - 我们来看看原来这些参数被设定为了多少 {% codeblock lang:xml line_number:false %}
      <Connector port="8080" protocol="HTTP/1.1"
      connectionTimeout="20000"
      redirectPort="8443" />
      {% endcodeblock %} 没被指定，全为默认，我们改成这样： {% codeblock lang:xml line_number:false %}
      <Connector port="8080" protocol="HTTP/1.1"
      connectionTimeout="5000"
      redirectPort="8443"
      maxConnections="10000"
      acceptCount="5000" />
      {% endcodeblock %}
        * 再重新运行攻击脚本，参数不变。 {% asset_img q1-2.png %}
        * 服务器一定程度上还是顶住了一小下，但随着连接建立越来越多，还是撑不住了，CPU处理率很快再次爆满。这种方法是治标不治本的。

2. 网站被攻击后，应该如何分析日志确定攻击者的IP呢？
    * 一般网站搭建软件都会提供访问日志记录功能，在类似于`/logs`的目录留下log文本文件。Tomcat也提供了这样的机制，在`安装目录/logs/`下存在着访问日志文件`<域名>_access_log.<date>.log`
      ，其中第一列记录了访问者的IP地址，双短横线后，紧接着是访问时间，然后是访问方式、访问地址、协议、状态码和返回字节数。从下图中的文件中，可以看出，刚才`192.168.235.136`在短时间内对网站首页`/`
      进行了大量的`post`操作，占用了大量资源，而这正好是刚才发动Torshammer攻击的机器的IP。 {% asset_img q2-1.png %}
    * 这个记录格式与apache和nginx都很像。只不过少了`user-agent`和`referer`信息。可以通过修改`server.xml`选项，加上一些信息来看看。比如把`user-agent`和`referer`加上。
      {% codeblock server.xml lang:xml line_number:false %}
      <Valve className="org.apache.catalina.valves.AccessLogValve"
      directory="logs"  prefix="localhost_access_log." suffix=".txt"
      pattern="combined" resolveHosts="false"  />
      {% endcodeblock %}
        - 其中 `combined` 为 `%h %l %u %t "%r" %s %b "%{Referer}i" "%{User-Agent}i"`
    * 重新设置后，现在能抓到日志如下 {% asset_img q2-2.png %}
        - 显然，这个脚本使用了随机的`user-agent`，所以不可能像限制爬虫名字一样，通过限制`user-agent`来抵抗此种攻击

### 本网站用到的架构相关资料

* SSH(Struts2+Spring+Hibernate)框架[本部分资料来自https://blog.csdn.net/u014577487/article/details/85558131]
    - SSH 框架是由 struts2、spring、hibernate 三大框架组合起来的一套总框架。
    - 浏览器（或客户端）发送请求到服务器，先经过项目中 `web.xml` 中过滤器（`<filter>` 和 `<filter-mapping>`）审核，通过了再发送给 `action` 包中的 `IndexAction`
      类，`struts.xml` 根据 `IndexAction` 类中 `return` 的值再进行跳转，跳转的页面是 `struts.xml` 中 `<result>` 配置的页面名，然后页面响应回客户端。
    - Struts是一个 Java Web MVC 开发框架。
        * 模型 Model 用于封装与业务逻辑相关的数据和数据处理方法
        * 视图 View 是数据的 HTML 展现
        * 控制器 Controller 负责响应请求，协调 Model 和 View
        * Model，View 和 Controller 的分开，是一种典型的关注点分离的思想，不仅使得代码复用性和组织性更好，使得 Web 应用的配置性和灵活性更好。
    - Spring 的核心思想：解耦，也就是代码中不出现 new 实现类的代码，我们创建了接口不用关心实现类是谁，实现类由 spring 帮我们注入，我们只需要在定义接口的时候给它一个 set
      方法并且在配置文件里改 `<property>` 中的 id 和 ref 就行
    - Hibernate 的核心思想：(ORM - 对象关系映射) 连接数据库，我们不用在数据库写创建表的语句，数据库表的字段根据实体类中属性的名字然后我们在 `*.hbm.xml` 文件里配置 `<property>`
      以及 `<property>` 的相关属性。
