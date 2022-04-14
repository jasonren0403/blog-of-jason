---
title: PHP网站搭建和SQLMAP工具使用 
date: 2019-10-24 17:22:18 
tags:
  - "网络空间安全"
  - "信息系统安全"
categories:
  - ["实验","信息系统"]
  - ["漏洞","SQL注入"]

---

## 实验目的

1. 了解搭建网站的一般流程，学习使用PHPStudy来搭建网站。
2. 了解SQLMap工具的原理和应用，利用sqlmap寻找数据库的注入漏洞。

<!-- more -->

## 实验内容

1. 使用PhpStudy和金戈企业建站框架搭建网站服务。
2. 使用sqlmap工具扫描该网站，寻找sql漏洞利用点。

## 实验环境

{% asset_img env.jpg %}

## 实验要求

{% tabs requirements, 1 %}
<!-- tab PHPStudy搭建网站 -->

1. 将服务器的端口更换为8080，域名修改为<code>www.test.com</code>。网站搭建成功后，通过在浏览器中访问<code>www.test.com</code>访问这个网站。
2. 通过端口扫描工具扫描整个网站所开设的端口。看网站是否在8080端口提供服务。

<!-- endtab -->
<!-- tab Sqlmap实验 -->
根据实验指导书进行操作
<!-- endtab -->
{% endtabs %}

## 实验步骤和结果

### 网站准备

1. 安装PHPStudy
    * 点击桌面上的phpstudysetup安装包，并为其指定一个合适的解压位置。等待其安装完毕 {% grouppicture 3-1 %} {% asset_img 建站1.png %} {% asset_img 建站2.png
      %} {% asset_img 建站3.png %} {% endgrouppicture %}

   {% asset_img 建站4.png "PHPStudy主界面" %}

2. 配置PHPStudy
    * 设置好网站目录、端口，然后点击“应用”。Apache会重启 {% grouppicture 4-3 %} {% asset_img 建站5.png "设置端口" %} {% asset_img 建站6.png "设置网站目录"
      %} {% asset_img 建站7.png "Apache部分设置全览" %} {% asset_img 建站8.png %} {% endgrouppicture %}

3. 配置域名
    1. 从主界面中进入站点域名管理界面。 {% asset_img 建站9.png "域名管理界面" %}
    2. 配置好域名后，点击“新增”。 {% asset_img 建站10.png %} {% asset_img 建站11.png "左侧“站点”栏中出现了新建的站点" %}
    3. 修改hosts文件，将刚刚起好的域名解析到本地 {% asset_img 建站12.png "添加一行127.0.0.1 <域名>" %}
    4. 测试访问效果 {% asset_img 建站13.png %}

4. 将网站切换到8080端口上
    1. 打开域名管理界面，把域名改为要求的`www.test.com`，端口设为`8080`，然后点击修改。 {% asset_img 建站14.png %} {% asset_img 建站15.png "要修改的项目" %}
    2. 相应的host中的域名要变一下。 {% asset_img 建站16.png "添加或修改成127.0.0.1 www.test.com" %}
    3. Apache中的httpd端口也要修改。 {% asset_img 建站17.png "改为8080" %}
    4. 此时再直接输入`www.test.com`就进不去了，得加上端口号才可以访问那个网站。 {% grouppicture 2-2 %} {% asset_img 建站18.png %} {% asset_img 建站19.png
       %} {% endgrouppicture %}
    5. 此时进行端口扫描，会发现网站的8080端口是开启的。 {% note default %} 192.168.235.138是web服务器的ip地址 {% endnote %} {% asset_img 建站20.png %}

### SQLMap初使用

{% note info %} Sqlmap是一款由Python语言开发的自动化SQL注入工具，其主要功能是扫描、发现并利用给定的URL的SQL注入漏洞。

本次我们使用Sqlmap工具寻找前一步搭建好的网站系统的SQL注入漏洞。 {% endnote %}

1. 首先找到一处注入点`ry.php?ry_id=?`。用引号测试下GET参数的情况。发现返回了空白页面而没有报错。推断可能存在注入漏洞 {% grouppicture 2-2 %} {% asset_img 注入1.png %} {%
   asset_img 注入2.png %} {% endgrouppicture %}

2. 上工具，首先测试该链接有无注入漏洞，用`-u url`命令 {% codeblock lang:bash line_number:false %} sqlmap -u <url>
   {% endcodeblock %} {% grouppicture 3-1 %} {% asset_img 注入3.png %} {% asset_img 注入4.png %} {% asset_img 注入5.png %} {%
   endgrouppicture %}

3. SQLmap工具输出表明`ry.php`页面存在GET参数`ry_id`
   注入漏洞，基于报错的注入，基于时间的盲注，联合注入方法均有效。还探测到了服务器的后端使用了php5.4.45，Apache2.4.23，使用MySQL数据库版本高于4.1，操作系统为Windows。 {% asset_img
   注入6.png %}

4. 接下来使用 `–dbs`选项来查找在该服务器上运行的所有数据库的名称列表。相当于执行了一次`show databases`的命令。 {% codeblock lang:bash line_number:false %} sqlmap
   -u <url> --dbs {% endcodeblock %} {% asset_img 注入7.png %}

    * 找到了5个数据库。对照服务器上的数据库列表，发现确实有这些数据库。 {% grouppicture 2-2 %} {% asset_img 注入8.png "sqlmap跑出来的数据库列表" %} {% asset_img
      注入9.png "本地工具截取出来的" %} {% endgrouppicture %}

5. 接下来，使用`--current-user`选项查找当前数据库的用户。 {% codeblock lang:bash line_number:false %} sqlmap -u <url> --current-user {%
   endcodeblock %} {% asset_img 注入10.png %}

    * 是`root@localhost`，权限相当大，可以想象这个数据库中拥有相当重要的数据。 {% asset_img 注入11.png %}

6. 用`--tables`选项查看`jnng`数据库中的所有表。相当于执行了一次`show tables`命令。`-D`选项确定数据库名称。 {% codeblock lang:bash line_number:false %}
   sqlmap -u <url> -D jnng --tables {% endcodeblock %} {% asset_img 注入12.png %} {% asset_img 注入13.png %}

    * root表应该有比较有趣的东西。一会可以看看。

7. 用`--columns`选项列出表的字段。`-T`确定表名。 {% codeblock lang:bash line_number:false %} sqlmap -u <url> -D jnng -T root --columns
   {% endcodeblock %} {% asset_img 注入14.png %} {% asset_img 注入15.png %}

    * 发现了疑似用户名和密码的字段`root_name`和`root_pass`，接下来我们要拿到这几个字段。

8. 使用--dump选项导出字段。-C选项确定要导出的字段名。SQLmap还会自动发现密码hash字段并自动使用内建的字典进行破解。 {% codeblock lang:bash line_number:false %} sqlmap
   -u <url> -D jnng -T root -C root_id,root_name,root_pass --dump {% endcodeblock %} {% asset_img 注入16.png %} {%
   grouppicture 2-2 %} {% asset_img 注入17.png %} {% asset_img 注入18.png %} {% endgrouppicture %}

9. 使用密码登录后台`www.test.com/isadmin/login.php`，发现可以登录，证明前一步从数据库取得的数据是正确的。
   {% note warning %} 后台网页显示有问题，但是没有报密码错误，证明可以登上 {% endnote %} {% grouppicture 2-2 %} {% asset_img 注入19.png %} {%
   asset_img 注入20.png %} {% endgrouppicture %}

10. 看看别的地方 {% grouppicture 2-2 %} {% asset_img 注入21.png %} {% asset_img 注入22.png %} {% endgrouppicture %} {%
    grouppicture 2-2 %} {% asset_img 注入23.png %} {% asset_img 注入24.png %} {% endgrouppicture %}

## 附录（可选）

### 其他难点和坑点

1. phpstudy包含了php5，6和7三个版本，每个版本需要安装不同的VC运行库
    * php5.3、5.4和apache都是用vc9编译，电脑必须安装vc9运行库才能运行。
    * php5.5、5.6是vc11编译，如用php5.5、5.6必须安装vc11运行库。
    * php7.0、7.1是vc14编译，如用php7.0、7.1及以上版本必须安装vc14运行库。
2. 利用phpstudy所建的网站文件夹及路径不能带有任何空格。
3. 想要用别的虚拟机访问windows server服务器，需要在server端将防火墙配置好，具体是开启80端口、开启ICMPv4-in，此时能ping通就差不多可以了。
4. 这个网站还有XSS漏洞 {% asset_img vul_xss.png %}

### 怎么修复这个注入点

查看发生问题的`ry.php`源码。发现这样一段

```php ry.php linenos:false
$ry_id = $_GET['ry_id'];
$ry = root_alli(qyry,ry_id,$ry_id);
```

`root_alli`函数中直接进行了条件拼接

```php ry.php linenos:false
$result = $conn->query('SELECT * FROM '.$tab.' where '.$zn.'  = '.$in);
```

GET 的`ry_id`未经过滤直接传入`root_alli`，造成了SQL语句的拼接。

{% note success %} 解决方案：先过滤（`my_yz()`调用了`addslashes`——常用的过滤操作） {% codeblock lang:php mark:1 %} $ry_id = my_yz($ry_id);

if (!is_numeric($ry_id))
{ // error exit; } {% endcodeblock %} {% endnote %}
