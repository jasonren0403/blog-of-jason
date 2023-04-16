---
title: Tomcat安全性分析及安全配置 
date: 2019-12-6 15:32:26 
tags:
  - "网络空间安全"
  - "信息系统安全"
categories:
  - ["实验","信息系统"]
  - ["漏洞","Tomcat配置"]

---

## 实验目的

了解Tomcat存在的主要风险及问题。 了解并掌握Tomcat安全配置基本策略。

<!-- more -->

## 实验内容

访问Tomcat的后台，了解后台暴露存在的问题，并尝试进行安全配置来修复这些问题。

## 实验环境

运行Tomcat8.0.30的Windows Server2012。

## 实验要求

分析tomcat存在的风险，利用所学知识进行一些简单的安全配置。

## 实验步骤与结果

{% asset_img slug %}

1. 在tomcat中，除了根目录`/`以外，还存在`docs`和`manager`两个子目录，`docs`存放tomcat的帮助文档，而`manager`是对tomcat服务器的web配置端，权限很大，如果配置不力，会带来安全隐患，假如默认用户名和密码再未更改，会相当危险。
    {% asset_img 5.png %}

    {% note primary 提示 %}
    这个配置文件在tomcat安装目录的conf文件夹下，现有版本下，如果tomcat-users没有加注释，那么使用tomcat作为用户名和密码，在拥有manager-gui权限（之前版本叫做manager）的情况下，就可以通过<code>ip:8080/manager</code>访问tomcat的后台，具体如下。
    {% asset_img 6.png "默认密码" %}
    {% asset_img 3.png %}
    {% asset_img 4.png %} 
    {% endnote %}
2. Deploy这里更是漏洞连连，它甚至为我们提供了一个上传文件的地方，还可以自动部著，我们随便找一个文件，把它的后缀名改成`.war`就可以传上去，（但是还是推荐标准命令打出来的war包：`jar -cvf`）这个`.war`文件要是包含`jsp`木马，会引起不好的后果。

    {% asset_img 1.png "上传文件接口" %}
    {% asset_img 2.png "上传了shell" %}

    {% note primary %}
    传上去的文件会在webapps下自动解压，接下来就可以通过/shell访问里面的内容。
    {% endnote %}

## 附录（可选）

1. 如何禁止Tomcat列表显示文件？
    - 列表显示文件可以说又是一个暴露信息的漏洞，如果配置不当，假如网站主页文件坏掉，目录就会显示出来，造成非授权的浏览。
       {% asset_img 10.png "目录显示的不良后果" %}
    - 查看`web.xml`中对于`listings`的介绍，发现默认即为`false`。保持即可，如果不是，在`/conf/web.xml`中`<servlet>`子项中将`listings`设置为`false`即可。
       {% asset_img 7.png "listing配置项介绍" %} {% asset_img 8.png "保持这里是false即可" %}

2. 如何更改Tomcat服务器默认管理端口？
    - `/conf/server.xml`中改变`Connector`的port值

3. 如何禁用不必要的http方法？
    - 服务器的`/conf/web.xml`中，作如下设置
        * 更换标准
           {% codeblock lang:xml line_number:false %}
           <web-app xmlns="http://java.sun.com/xml/ns/j2ee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
           version="2.4">
           {% endcodeblock %}
        * 添加设置
           {% codeblock lang:xml line_number:false %}
           <security-constraint>
           <web-resource-collection>
           <url-pattern>/*</url-pattern>
           <http-method>PUT</http-method>
           <http-method>DELETE</http-method>
           <http-method>HEAD</http-method>
           <http-method>OPTIONS</http-method>
           <http-method>TRACE</http-method>
           </web-resource-collection>
           <auth-constraint>
           </auth-constraint>
           </security-constraint>
           <login-config>
           <auth-method>BASIC</auth-method>
           </login-config>
           {% endcodeblock %}
        * 或对应用程序的`web.xml`做第二部分设置 {% asset_img 12.png %}
    - 使用postman做测试：对首页发起PUT请求，发现返回错误信息，说明配置成功。 {% asset_img 11.png %}

4. 如何禁用webdav？
    {% note info no-icon 什么是webdav %}
    WebDAV（Web-based Distributed Authoring and Versioning）是一种基于 HTTP 1.1协议的通信协议.它扩展了HTTP 1.1，在GET、POST、HEAD等几个HTTP标准方法以外添加了一些新的方法，使应用程序可直接对Web Server直接读写，并支持写文件锁定(Locking)及解锁(Unlock)，还可以支持文件的版本控制。
    {% endnote %}

    HTTP/1.1协议中共定义了八种方法（有时也叫“动作”）来表明Request-URI指定的资源的不同操作方式：

    {% tabs http, 1 %}

	<!-- tab OPTIONS -->
	返回服务器针对特定资源所支持的HTTP请求方法。也可以利用向Web服务器发送'*'的请求来测试服务器的功能性。
	<!-- endtab -->
	<!-- tab HEAD -->
	HEAD 向服务器索要与GET请求相一致的响应，只不过响应体将不会被返回。这一方法可以在不必传输整个响应内容的情况下，就可以获取包含在响应消息头中的元信息。
	<!-- endtab -->
	<!-- tab GET-->
	GET 向特定的资源发出请求。注意：GET方法不应当被用于产生“副作用”的操作中，例如在web app中。其中一个原因是GET可能会被网络蜘蛛等随意访问。
	<!-- endtab -->
	<!-- tab POST -->
	POST 向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立和/或已有资源的修改。
	<!-- endtab -->
	<!-- tab PUT -->
	PUT 向指定资源位置上传其最新内容。
	<!-- endtab -->
	<!-- tab DELETE -->
	DELETE 请求服务器删除Request-URI所标识的资源。
	<!-- endtab -->
	<!-- tab TRACE -->
	TRACE 回显服务器收到的请求，主要用于测试或诊断。
	<!-- endtab -->
	<!-- tab CONNECT -->
	CONNECT HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。
	<!-- endtab -->
	{% endtabs %}

   * 方法名称是区分大小写的。当某个请求所针对的资源不支持对应的请求方法的时候，服务器应当返回状态码405（Method Not Allowed）；当服务器不认识或者不支持对应的请求方法的时候，应当返回状态码501（Not Implemented）。
   * HTTP服务器至少应该实现GET和HEAD方法，其他方法都是可选的。当然，所有的方法支持的实现都应当符合下述的方法各自的语义定义。此外，除了上述方法，特定的HTTP服务能够扩展自定义的方法。
   * HTTP的访问中，一般常用的两个方法是：GET和POST。其实主要是针对DELETE等方法的禁用。所以，本题答案与3一样
