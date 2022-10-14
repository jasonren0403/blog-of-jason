---
title: 数据控制
date: 2018-12-3 09:44:22
tags:
  - "数据库"
  - "MySQL"
categories:
  - ["实验", "数据库"]

---

## 实验目的
1. 熟悉通过SQL对数据库进行数据控制，包括安全性和完整性。
2. 熟悉数据库的备份与恢复操作。

<!-- more -->

## 实验环境
本次实验使用MySQL软件，使用其图形化界面完成权限的全套管理工作。备份与还原数据部分使用命令行工具完成。

所用数据库内容为计算机产品数据库，内容同 {% post_link experiments/database/interactive-sql "实验二" %}，实验环境与前两次实验完全相同。

## 实验内容和完成情况
### 实验准备
1. 从MySQL界面中选择root用户登录
    {% asset_img preparation_create_user.png %}

2. 在左侧管理栏中点击`Administration`，然后点击`Users and privileges`，在其中添加用户。

    {% asset_img preparation_create_user_2.png %}

    或者在命令行输入`create user`命令创建用户。
    {% asset_img preparation_create_user_3.png %}

{% note info %}
#### 将创建以下用户
* 2017522133（root+所有权限）
* U1~U7（创建用户时默认对其赋予`Connect`权限）
{% endnote %}
### 数据的安全性
1. 完成DBA方面的授权。包括：
    * 将查询PC表的权利授给用户U1；
	    ```sql
		grant select on table PC to U1;
	    ```
    * 把对PC表和Laptop表的全部操作权限赋予用户U2和U3；
        ```sql
		grant all privileges on table PC to U2;
        grant all privileges on table Laptop to U2;
        grant all privileges on table PC to U3;
        grant all privileges on table Laptop to U3;
		```
    * 把对Product表的查询权限赋予全体用户；
        ```sql
	    grant select on table Product to public;
	    ```
    * 把查询Laptop表和修改笔记本价格的权限赋予用户U4；
        ```sql
		grant update(price),select on table Laptop to U4;
        ```
    * 把对表Product的INSERT权限授予U5用户，并允许将此权限再授予其他用户。
        ```sql
        grant insert on table Product to U5 with grant option;
	    ```
2. U5将他获得的`INSERT`权限转给U6。
    1. 以U5身份登录数据库。
        {% asset_img grant_4.png %}
    2. 命令窗口中输入`grant insert on table Product to U6;`。
        {% asset_img grant_5.png %}
3. U6试图将Product表的Insert权限传播给U7。
	1. 以U6身份登录数据库。
	    {% asset_img grant_6.png %}
	2. 命令窗口中输入`grant insert on table Product to U7;`。
	    {% asset_img grant_7.png %}
    3. 运行失败，出现1142错误。其意义为：”U6不可使用`GRANT`命令传播product表的`INSERT`权限”。
	    {% asset_img grant_8.png %}
4. 收回用户U3对于PC表的查询权限。
	1. 以2017522133用户身份登录数据库，输入`revoke select on table PC from U3;`。
		{% asset_img revoke_1.png %}
	2. 以U3用户身份登录，试图在PC表中进行`SELECT`操作。
		{% asset_img revoke_2.png %}
    3. 操作失败，显示1142错误“U3对于PC表的`SELECT`操作被禁止”。
        {% asset_img revoke_3.png %}
5. 收回用户U5对于Product表的`INSERT`权限。
    1. 以root身份登录，输入`revoke insert on table Product from U5;`。
		{% asset_img revoke_4.png %}
    2. 以U5身份登录，试图进行插入操作，结果操作失败。
        {% asset_img revoke_5.png %}
        {% asset_img revoke_6.png %}
    3. 以U6身份登录，试图进行插入操作，结果操作成功。说明MySQL对于`REVOKE`语句的默认值为`RESTRICT`。
        {% asset_img revoke_7.png %}
        
        注意U6没有查询权限！需要登录root用户才能查询到下图数据。
        {% asset_img revoke_8.png %}
### 数据的完整性
已知三张表格的DDL语言如下：
{% tabs ddl %}
<!-- tab product -->
{% asset_img product_define.png %}
<!-- endtab -->
<!-- tab laptop -->
{% asset_img laptop_define.png %}
<!-- endtab -->
<!-- tab pc -->
{% asset_img pc_define.png %}
<!-- endtab -->
{% endtabs %}
1. [实体完整性I]在对pc表添加数据时没有指定model。
    {% gp 2-1 %}
    {% asset_img 5.1.1.png %}
    {% asset_img 5.1.1result.png %}
    {% endgp %}
2. [实体完整性II]在对product表添加数据时没有指定model和maker（这两个字段为product表主码）。
    先完成表级设定。
    ```sql
    ALTER TABLE `computer_products`.`product` 
    CHANGE COLUMN `maker` `maker` CHAR(2) NOT NULL,
    DROP PRIMARY KEY,
    ADD PRIMARY KEY (`model`, `maker`);
    ```
    {% note warning %}
    要先drop掉当前的primary key再添加，不然会出现重复定义错误。
    {% asset_img 5.1.1error.png %}
    {% endnote %}
    然后试图添加数据，失败。错误信息与前述相同。
    {% gp 2-1 %}
    {% asset_img 5.1.2.png %}
    {% asset_img 5.1.2result.png %}
    {% endgp %}
3. [参照完整性I]试图从product表中删除数据或在laptop表中插入数据，参照完整性定义为`RESTRICT`。
    {% asset_img laptop.png %}
    {% tabs product-laptop %}
    <!-- tab 子表插入数据 -->
    {% asset_img ref_1.png %}
    {% asset_img ref_1_error.png %}
    插入操作错误信息：
    ```
    Error Code: 1452. Cannot add or update a child row: a foreign key constraint fails (`computer_products`.`laptop`, CONSTRAINT `laptop_ibfk_1` FOREIGN KEY (`model`) REFERENCES `product` (`model`))
    ```
    <!-- endtab -->
    <!-- tab 父表删除数据 -->
    {% asset_img ref_2.png %}
    {% asset_img ref_2_error.png %}
    删除操作错误信息：
    ```
    Error Code: 1451. Cannot delete or update a parent row: a foreign key constraint fails (`computer_products`.`laptop`, CONSTRAINT `laptop_ibfk_1` FOREIGN KEY (`model`) REFERENCES `product` (`model`))
    ```
    <!-- endtab -->
    {% endtabs %}
4. [参照完整性II]试图从product表中删除数据或在pc表中插入数据，参照完整性定义为`CASCADE`。
    {% asset_img pc.png %}
    {% tabs product-pc %}
    <!-- tab 子表插入数据 -->
    {% asset_img ref_3.png %}
    {% asset_img ref_3_error.png %}
    插入操作错误信息：
    ```
    Error Code: 1452. Cannot add or update a child row: a foreign key constraint fails (`computer_products`.`pc`, CONSTRAINT `pc_ibfk_1` FOREIGN KEY (`model`) REFERENCES `product` (`model`) ON DELETE CASCADE ON UPDATE CASCADE)
    ```
    <!-- endtab -->
    <!-- tab 父表删除数据 -->
    {% asset_img ref_4.png %}
    {% gp 2-2 %}
    {% asset_img ref_4_result1.png %}
    {% asset_img ref_4_result2.png %}
    {% endgp %}
    可以删除，且pc表中的相关数据也被直接删除。
    <!-- endtab -->
    {% endtabs %}
5. [参照完整性III]试图从product表中删除数据或在laptop表中插入数据，参照完整性定义为`SET NULL`。
    * 本例无法完成，因为product表的主码和laptop表的主码一致且不能为空。
6. [用户定义完整性]Product表的形式定义语言如下
    {% asset_img ref_5.png %}
    1. Model被设置为列值唯一，不能再插入相同的值。
        {% asset_img ref_5_2.png %}
        {% asset_img ref_5_error.png %}
    2. (`CHECK`短语)临时表Computers定义如下
        ```sql
        create table Computers
	    (maker char(2) check(maker in('A','B','C')), //厂商只有A、B、C三家
	    model int(4),
	    price int(5) check(price>=1000 and price<=10000), //所有电脑的价格在1000元到10000元之间
	    primary key(model),
	    check (maker='B' or price<=5000)   //A、C两厂家的电脑价格都在5000元以下
	    );
        ```
        
        试图进行插入操作。
        {% asset_img ref_6_0.png %}
        这些值均插入成功，MySQL 表格定义的CHECK字句没能起作用！
        {% asset_img ref_6_1.png %}
    3. (`CONSTRAINT`短语)临时表Computers重新定义如下
        {% asset_img ref_7_0.png %}
        代码无法通过编译，应该是MySQL的语法与SQL语言标准不同的原因导致不能完全按课本示例来创建。

### 数据的备份与恢复
{% note warning %}
请检查binlog是否开启，开启方法见本文[最后](#遇到的问题和解决方法)。
{% asset_img backup_0.png %}
{% endnote %}

1. 创建备份设备，本例利用`mysqldump`命令将computer_products数据库放在了E盘根目录中。此操作需要输入密码。
	```sql
	mysqldump -u 2017522133 -p computer_products>E:\computer_products.sql
	```
	{% gp 2-2 %}
	{% asset_img backup_1.png %}
	{% asset_img backup_2.png %}
	{% endgp %}

	{% note info %}
    这个SQL文件可以用MySQL图形界面打开，内容大概如下。
    {% asset_img backup_3.png "实际上就是重建表格的过程" %}
	{% endnote %}

2. 对表格做插入操作。
    {% asset_img backup_4.png %}
3. 先执行`show master status`，然后执行`flush logs`，将`master status`显示的文件复制到其他地方。执行`flush logs`意义在于切换为新的日志文件，将问题段集中在一个日志文件中。
    {% asset_img backup_5.png %}
	{% asset_img backup_6.png %}

    binlog.000011文件内容如下，可以看到，从pos=4到pos=795经历了两次commit操作，分别为两条数据的新建操作，经分析，若要恢复数据新建的状态，只需恢复这个log即可。
	{% asset_img backup_7.png %}

4. 执行`source E:/computer_products.sql`。经过一段操作后，登录数据库，查看表格。
	{% asset_img backup_8.png %}
    下图表明数据库成功恢复到了数据插入前的状态。
    {% asset_img backup_9(4.5).png %}
5. 接下来使用`mysqlbinlog`命令恢复插入两条数据后的状态，此命令需要在Windows cmd命令行下进行。
	```sql
	mysqlbinlog E:/back_binlog.000011 | mysql -u <我的用户名> -p <我的密码> --database=computer_products
	```
	根据提示输入管理员密码。
	{% asset_img backup_10.png %}
6. 登录数据库查看表格情况。下图显示表明，数据库已恢复至插入数据后的状态。
	{% gp 2-2 %}
	{% asset_img backup_11.png %}
	{% asset_img backup_12.png %}
	{% endgp %}
## 遇到的问题和解决方法
1. 授权出错，显示 `You are not allowed to create a user with GRANT`。

	原因：在网上有很多教程说当出现`The user specified as a definer ('root'@'%') does not exist`时表示`root`用户权限不足，只需要执行`GRANT ALL ON *.* TO 'root'@'%';`就可以了，但是往往又会出现`You are not allowed to create a user with GRANT`的错误提示。这是因为`GRANT ALL ON *.* TO 'root'@'%';`这条语句中`@'%'`中的百分号其实是root用户对应host的名称，很多人并没有注意到他的root用户对应的其实是localhost，直接就执行了上面的语句，所以才会报错。
	
	解决方案：只要将`GRANT ALL ON *.* TO 'root'@'%';`中的`%`改为对应的`host`名称即可，最后还要刷新一下权限`FLUSH PRIVILEGES;`。

2. MySQL中的角色（Roles）概念？不是没有角色吗？

	在MySQL8.0版本后，有了Roles概念的引入。MySQL的官方文本对此作了介绍。其中明确提出——
	
	{% blockquote MySQL Dev Doc https://dev.mysql.com/doc/refman/8.0/en/roles.html#roles-creating-granting %}
	1. 可以用`CREATE ROLE`命令建立角色；
	2. 可以用`GRANT`命令授予这些角色以权限；
	3. 可以用`SHOW GRANTS FOR`命令检查某用户拥有的权限；
	4. 可以用`REVOKE`命令收回用户的权限；
	5. 可以用`DROP ROLE`命令删除角色。
	{% endblockquote %}

3. MySQL开启binlog的方法。
	* 打开MySQL配置文件`my.cnf`，在[mysqld]下面增加`log-bin=mysql-bin`（UNIX）
	* 修改MySQL配置文件`my.ini` ，添加配置：`log-bin=log-bin=<日志存储实际路径>`（WIndows）
	* 把前面的`#`去掉。
4. MySQL中备份与恢复命令语法
   * `mysqldump`：用于做数据库的全量备份。
   格式：`mysqldump -h主机名  -P端口 -u用户名 -p密码 –database 数据库名 > 文件名.sql`
   * `source`：用于从全量备份中恢复。
   用use进入到某个数据库，`mysql>source d:\test.sql`（刚备份的文件）
   * `mysqlbinlog`：用于查看binlog以及从日志文件恢复。
   
   * 查看日志：
       1. 查看所有binlog日志列表
           ```sql
           mysql> show master logs;
           ```
       2. 查看master状态，即最后(最新)一个binlog日志的编号名称，及其最后一个操作事件pos结束点(Position)值
           ```sql 
           mysql> show master status;
           ```
       3. 刷新log日志，自此刻开始产生一个新编号的binlog日志文件
           ```sql
           mysql> flush logs;
           ```
		  {% note warning %}
          每当mysqld服务重启时，会自动执行此命令，刷新binlog日志；在`mysqldump`备份数据时加 `-F` 选项也会刷新binlog日志。
          {% endnote %}
       4. 重置(清空)所有binlog日志
           ```sql
           mysql> reset master;
           ```
       5. 从日志中恢复：
           ```sql
	       mysqlbinlog mysql-bin.0000xx | mysql -u用户名 -p密码 数据库名
	       ```

           {% tabs mysqlbinlog-option %}
           <!-- tab 常用选项 -->
           ```
           --start-position=953                   起始pos点
           --stop-position=1437                   结束pos点
           --start-datetime="start_date" 起始时间点
           --stop-datetime="stop_date"  结束时间点
           --database=x                   指定只恢复x数据库(一台主机上往往有多个数据库，只限本地log日志)
           ```
           <!-- endtab -->
		   <!-- tab 不常用选项 -->
		   ```
           -u --user=name              Connect to the remote server as username.连接到远程主机的用户名
	       -p --password[=name]        Password to connect to remote server.连接到远程主机的密码
	       -h --host=name              Get the binlog from server.从远程主机上获取binlog日志
	       --read-from-remote-server   Read binary logs from a MySQL server.从某个MySQL服务器上读取binlog日志
	       ```
		   <!-- endtab -->
           {% endtabs %}

## 资料来源
- https://blog.csdn.net/nuli888/article/details/52117646
- https://www.cnblogs.com/Cherie/p/3309456.html
