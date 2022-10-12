---
title: 认识DBMS
date: 2018-11-8 10:08:06
tags:
  - "数据库"
  - "MySQL"
categories:
  - ["实验", "数据库"]

---

## 实验目的

通过安装和简单使用MySQL，熟悉数据库管理软件的基本使用方法。

<!-- more -->

## 实验环境

{% asset_img 环境.png %}
本次实验使用 MySQL Community 8.0.12.0版。操作系统如上图所示。

## 实验内容与完成情况

### 安装

1. 安装界面如图所示
    {% asset_img 1.png %}
2. 选择安装类型，这里选择 Developer Default套装，根据描述，将装入以下套件：
	* MySQL Server
	* MySQL Shell
	* MySQL Router
	* MySQL Workbench
	* MySQL for Excel
	* MySQL for Visual Studio
	* MySQL Connectors
	* Examples and tutorials
	* Documentation
	  {% asset_img 2.png %}
3. 配置认证方式和密码强度，保持默认选项
    {% asset_img 3.png %}
4. 配置root账户密码，*一定要记住*。
    {% asset_img 5.png %}
    这里也可以配置DB账户并赋予相应权限，如下图
    {% asset_img 4.png %}
5. 在配置服务时，最好勾选上这两项，这样开机时就不需要再手动启动MySQL服务。后面的安装过程就没有什么要说的了。
    {% asset_img 6.png %}

### 创建用户，赋予权限

{% asset_img 7.png %}

1. 启动MySQL Workbench软件，主界面如上图。
2. 在主界面点击Local instance MySQL80登录到Server上。登录后界面如下图。
    {% asset_img 8.png %}
3. 在Management栏中，选择Users and Privileges。然后点击Add Account。
    {% note info %}
    名为2017522133的账户刚才已经在安装过程中创建了
    {% endnote %}
    {% asset_img user_conf_1.png %}
4. 在Administrative Roles中，勾选DBA，发现后面几项同样被勾选。然后点击Apply。
    {% asset_img user_conf_2.png "现在这个账户就具有了DBA权限" %}

### 服务的启动与终止

1. 单击左侧Instance栏中的Startup/Shutdown，会见到如下界面。现在数据库服务是停止状态。
    {% asset_img server_stopped.png %}
2. 点击Start Server，现在数据库服务是运行状态。
    {% asset_img server_running.png %}
3. 在数据库服务运行的同时，可以点击Bring Offline，在不停止服务的情况下让数据库离线运行。
    {% asset_img server_offline.png %}

### 创建数据库和表

1. 点击菜单中的File，选择New Model，进入如下界面，再点击Add Table开始添加表格。
    {% asset_img 9.png %}
2. Columns栏用于定义列名称和数据类型，主码和Not Null要求也将在这里指定。
    {% asset_img 10.png %}
    {% note info %}
    特别地，若勾选Primary Key，则Not Null也被同时勾选。符合主码非空的原则。
    {% endnote %}
3. Foreign Keys用于设置外键，可以置入参照，例如例子中的Course表和SC表。
    {% grouppicture 2-2 %}
    {% asset_img 11.png %}
    {% asset_img 12.png %}
    {% endgrouppicture %}
4. 在Inserts中为表格添加内容，以下是建好的表格内容。
    {% tabs table-and-schemas %}
    <!-- tab Course表 -->
    {% grouppicture 2-2 %}
    {% asset_img Course-schema.png "Course表结构" %}
    {% asset_img Course-data.png "Course表数据" %}
    {% endgrouppicture %}
    <!-- endtab -->
    <!-- tab Student表 -->
    {% grouppicture 2-2 %}
    {% asset_img Student-schema.png "Student表结构" %}
    {% asset_img Student-data.png "Student表数据" %}
    {% endgrouppicture %}
    <!-- endtab -->
    <!-- tab 选课关系——SC表 -->
    {% grouppicture 2-2 %}
    {% asset_img SC-schema.png "SC表结构" %}
    {% asset_img SC-data.png "SC表数据" %}
    {% endgrouppicture %}
    <!-- endtab -->
    {% endtabs %}

## 实验总结

通过本次实验，我了解到了MySQL中的一些基本操作，包括安装、启动/停止服务、创建基本表等操作，并创建了一个简单的学生选课表。在之后的实验中，将使用SQL语句进行表格查询的工作。