---
title: SOC平台组建 
date: 2019-12-17 13:39:51 
tags:
  - "网络空间安全"
  - "信息系统安全"
categories:
  - ["实验","信息系统"]

---

## 实验目的

掌握在Ubuntu系统中配置java环境的方法，学习配置一个日志服务器。
<!-- more -->

## 实验内容

1. 使用Ubuntu18.04系统，在系统中安装Java开发环境
2. 配置日志服务器

## 实验环境

{% asset_img system.png "所用为Ubuntu系统" %}

## 实验要求

同（二）实验内容

## 实验步骤和结果

### 安装JDK

1. 本次选择将jdk8安装在`/usr/local`文件夹下。 {% asset_img 1.png %}
2. 获取`root`权限，并解压jdk压缩包。 {% asset_img 2.png %}
3. 使用gedit打开`/etc/profile`文件，在最后四行加入Java环境变量。利用`source`命令让profile文件生效。
    {% codeblock lang:bash %}
    export JAVA_HOME=/usr/local/jdk_1.8.0_191
    export JRE_HOME=${JAVA_HOME}/jre
    export CLASSPATH=.:$[JAVA_HOME]/lib:${JRE_HOME}/lib
    export PATH=${JAVA_HOME}/bin:$PATH
    {% endcodeblock %}
    {% asset_img 3.png %} {% asset_img 4.png %}
4. 命令行中输入`java -version`，测试安装效果和环境变量是否设置正确。 {% asset_img 5.png %}

### 配置SOC日志服务器

{% tabs soc,1 %}
<!-- tab 服务端 -->

1. 编辑服务器端的`/etc/rsyslog.conf`文件，取消`imtcp`相关的注释并加入如下设置。 {% asset_img 6.png %}
2. 查看服务端IP，并开启rsyslog服务。
    {% grouppicture 2-2 %}
    {% asset_img 8.png "服务端IP" %}
    {% asset_img 7.png "开启服务" %}
    {% endgrouppicture %}

<!-- endtab -->
<!-- tab 客户端 -->

1. 编辑客户端的`rsyslog.conf`文件，作如下设置。
    {% asset_img 10.png %}
    {% note info %} 
    其中192.168.235.146是刚才服务器的地址。
    {% endnote %}
2. 查看客户端IP并启动rsyslog服务。
    {% grouppicture 2-2 %}
    {% asset_img 9.png "客户端IP" %}
    {% asset_img 11.png "开启服务" %}
    {% endgrouppicture %}

<!-- endtab -->
{% endtabs %}

1. 在client端利用logger命令打日志，测试如下 {% asset_img 12.png %}
2. 服务器端可以在`/var/log`下找到当前日期命名的log文件，其内容如下 {% asset_img 13.png %}
3. 说明带有`test_log`标志的日志被成功记录，设置生效。
4. 打一条不带`test_log`标志的记录，根据预期规则设置，此条log应该被丢弃。 {% asset_img 14.png %}
5. 果然没有相应记录，规则生效。 {% asset_img 15.png %}

## 附录（可选）

思考题回答：

1. ubuntu中如何设置文件权限？
    * `chmod`命令，根据命令帮助手册说明，我们可以使用`chmod [八进制权限标识|权限赋值表达式] 文件1[,文件2,…]`来改变某文件的权限。
       {% grouppicture 2-2 %}
       {% asset_img set-priv.png "八进制权限标识" %}
       {% asset_img set-priv2.png "权限赋值表达式" %}
       {% endgrouppicture %}
    
       {% note info %}
       权限标识中，`r`代表读权限，值为4；`w`代表写权限，值为2；`x`代表执行权限，值为1。权限赋值表达式中，可以使用`+|-|=`三种运算符为某一组用户赋值。
       {% endnote %}

2. 如何关闭rsyslog服务？
    * 使用与开启rsyslog相似的命令（把`start`换成`stop`）就可以关闭rsyslog服务，如果以非root用户身份运行，需要认证密码。 {% asset_img stop-1.png %}
    * 注意到上方输出中含有“via systemctl”的字样，通过查阅资料得知，Ubuntu Linux在关闭服务时调用了`systemd`工具下的`systemctl`，所以使用`systemctl`相关命令，应该有类似的效果。
       {% grouppicture 2-2 %}
       {% asset_img stop-2.png "systemctl stop 服务名" %}
       {% asset_img stop-3.png "service stop 服务名" %}
       {% endgrouppicture %}
