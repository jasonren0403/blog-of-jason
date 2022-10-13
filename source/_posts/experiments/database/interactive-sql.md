---
title: 交互式SQL
date: 2018-11-17 22:12:27
tags:
  - "数据库"
  - "MySQL"
categories:
  - ["实验", "数据库"]

---

## 实验目的
通过进行给定使用场景下的SQL语句编写和运行，理解SQL语句的执行过程。

<!-- more -->

## 实验环境
* 本次实验使用MySQL Community 8.0.12.0版。
* 本次实验使用了MySQL Workbench工具。
* 操作系统同 {% post_link experiments/database/hello-dbms "实验一" %}。

## 实验内容和完成情况
### 本次使用的数据库内容
{% tabs tables %}
<!-- tab Product -->

|maker|model|type|
|:--|:--|:--|
|A|1001|pc|
|A|1002|pc|
|A|2004|laptop|
|B|1003|pc|
|B|1004|pc|
|B|2005|laptop|
|C|1005|pc|
|C|2006|laptop|
|C|2007|laptop|

<!-- endtab -->
<!-- tab PC -->

|model|speed|ram|hd|price|
|:--|:--|:--|:--|:--|
|1001|2.66|1024|250|2114|
|1002|2.10|512|250|995|
|1003|1.42|512|80|478|
|1004|2.80|1024|250|649|
|1005|3.15|2048|500|4028|

<!-- endtab -->
<!-- tab Laptop -->

|model|speed|ram|hd|screen|price|
|:--|:--|:--|:--|:--|:--|
|2004|2.00|512|60|13.3|1150|
|2005|2.16|1024|120|17.0|2500|
|2006|5.00|4096|500|17.0|6094|
|2007|3.58|4028|120|14.5|4000|

<!-- endtab -->
{% endtabs %}

### 数据定义
#### 模式的创建与删除
1. 为当前用户创建一个名为`computer_products`的模式。
    ```sql
	CREATE SCHEMA `computer_products`;
	```

    {% grouppicture 4-3 %}
    {% asset_img create_schema_1.png %}
    {% asset_img create_schema_2.png %}
    {% asset_img create_schema_3.png %}
    {% asset_img create_schema_4或双击左侧列表中的模式名.png %}
    {% endgrouppicture %}

	{% note info %}
    1. MySQL创建模式时不需要`AUTHORIZATION`子句。
    2. 创建完模式后，需要双击`computer_products`或使用`use computer_products;`命令进入该模式。
    3. 应当进行已有数据库的检查，若已存在，则先删除再插入。其实采用以下语句会更加科学。
		```sql
		DROP SCHEMA IF EXISTS `computer_products`;
		CREATE SCHEMA IF NOT EXISTS `computer_products`;
		```
    {% endnote %}

2. 本次实验已结束，本模式不再需要。请删除`computer_products`模式。
	```sql
	DROP SCHEMA computer_products;
	```

	{% grouppicture 3-3 %}
	{% asset_img drop_schema_1.png %}
	{% asset_img drop_schema_2.png %}
	{% asset_img drop_schema_3.png %}
	{% endgrouppicture %}

	执行成功后，SCHEMAS栏中不再有此模式（见最右图）。

#### 基本表的创建与删除
1. 创建计算机产品数据库的 Product 表、PC表和Laptop表。
    
    {% tabs table-ddl %}
    <!-- tab Product -->
    ```sql
	create table Product(
	   maker char(2), 
	   model int(4) primary key,
	   type char(10)
    );
	```
	<!-- endtab -->
	
    <!-- tab PC -->
	```sql
    create table PC(
	  model int(4) primary key,
	  speed float(3),
	  ram int(5),
	  hd int(4),
	  price int(5),
	  foreign key (model) references Product(model)
	);
    ```
	<!-- endtab -->

	<!-- tab Laptop -->
	
	```sql
    create table Laptop(
	  model int(4) primary key,
	  speed float(3),
	  ram int(5),
	  hd int(4),
	  screen float(2),
	  price int(5),
	  foreign key (model) references Product(model)
	);
    ```
	<!-- endtab -->
	{% endtabs %}

    {% gp 6-4 %}
    {% asset_img create_table1.png %}
    {% asset_img create_table1_select.png %}
    {% asset_img create_table2.png %}
    {% asset_img create_table2_select.png %}
    {% asset_img create_table3.png %}
    {% asset_img create_table3_select.png %}
    {% endgp %}

    {% asset_img create_table_result.png "创建结果" %}

	{% note info %}
    1. MySQL中引用外码的写法是`FOREIGN KEY (<外码列名>) REFERENCES <表名>(<引用列名>)`。
    2. 后来，我将screen和speed的数据类型改为了`decimal`（MySQL中不支持numeric的写法），在后面的数据查找操作中会简单讲下为什么要这样做。
    {% endnote %}
2. 为PC表中加入屏幕尺寸列，取值为一位小数。
	```sql
	ALTER TABLE pc ADD COLUMN screen FLOAT(1);
	```
    {% gp 2-2 %}
    {% asset_img alter_table_1.png %}
    {% asset_img alter_table_2.png %}
	{% endgp %}
3. 为统计屏幕尺寸创建了一个新表screen，现在统计结束，不再需要此表。
	```sql
	DROP TABLE screen;
	```
 
	{% gp 3-3 %}
	{% asset_img drop_table_1.png %}
	{% asset_img drop_table_2.png %}
	{% asset_img drop_table_3.png %}
	{% endgp %}

#### 索引的建立与删除 
1. 按照降序序列建立PC表中价格的索引。
	```sql
	CREATE INDEX pc_price ON pc(price DESC);
	```
 
	{% gp 3-3 %}
	{% asset_img create_index_0.png %}
	{% asset_img create_index_1.png %}
	{% asset_img create_index_2.png %}
	{% endgp %}

2. 1中的索引不再需要，请删除`pc_price`索引。
	```sql
	DROP INDEX pc_price ON pc;
	```
 
	{% gp 2-2 %}
	{% asset_img drop_index_1.png %}
	{% asset_img drop_index_2.png %}
	{% endgp %}

	{% note info %}
    在MySQL中，`drop index`语句中也有`on`子句，需要指定索引所在表。
   	{% endnote %}

### 数据操作
#### 数据更新
1. 向计算机产品数据库中添加给定数据。
	{% gp 6-4 %}
	{% asset_img insert_into_table1.png %}
	{% asset_img insert_into_table1_select.png %}
	{% asset_img insert_into_table2.png %}
	{% asset_img insert_into_table2_select.png %}
	{% asset_img insert_into_table3.png %}
	{% asset_img insert_into_table3_select.png %}
	{% endgp %}
	
	{% note info %}
    在 MySQL 中，`VALUES`子句可以一次插入多组值。
	{% endnote %}

2. 将所有硬盘容量为120GB的笔记本的价格下调200元。
	```sql
	UPDATE laptop
	SET price=price-200
	WHERE hd=120;
	```

	{% gp 2-2 %}
	{% asset_img update_data_1.png %}
	{% asset_img update_data_2.png %}
	{% endgp %}

3. 型号为2004的电脑已经停产，请删除其相关记录。
	```sql
	DELETE
	FROM laptop
	WHERE model=2004;
	```
 
	{% gp 2-2 %}
	{% asset_img delete_data_1.png %}
	{% asset_img delete_data_2.png %}
	{% endgp %}

#### 数据查询
1. （单表查询）查询制造商A的所有产品，按照产品序列号降序排列。
    ```sql
    SELECT * FROM product
    where maker='A'
    ORDER BY model DESC;
    ```

   {% gp 2-2 %}
   {% asset_img select_1.png %}
   {% asset_img select_1_result.png %}
   {% endgp %}

2. （连接查询I）查询生产了速度为2.80的PC机制造商。
    ```sql
    SELECT maker FROM product, pc
    WHERE product.model=pc.model
    AND speed=2.8;
    ```
 
    {% gp 2-2 %}
	{% asset_img select_2.png %}
	{% asset_img select_2_result.png %}
    {% endgp %}

	{% note warning %}
    由于浮点数比较问题，这里的speed若是float或double类型，则会查找失败！
    解决方案：将speed设为decimal(dec)类型。
	{% endnote %}

3. （连接查询II）计算制造商C生产的所有笔记本的总价格。
    ```sql
    SELECT SUM(price)
    FROM product, laptop
    WHERE maker IN('C')
    AND product.model=laptop.model;
    ```

   {% gp 2-2 %}
   {% asset_img select_3.png %}
   {% asset_img select_3_result.png %}
   {% endgp %}

4. （嵌套查询I）查询生产了型号为2007号笔记本的厂商都生产了哪些产品和型号。
    ```sql
    SELECT model, type
    FROM product
    WHERE maker IN (SELECT maker
            FROM product, pc, laptop
            WHERE (product.model=pc.model OR product.model=laptop.model)
            AND product.model=2007);
    ```

   {% gp 2-2 %}
   {% asset_img select_4.png %}
   {% asset_img select_4_result.png %}
   {% endgp %}

5. （嵌套查询II）查询硬盘容量比所有PC机都小的笔记本电脑的型号及其硬盘容量大小。
    ```sql
    SELECT model, hd
    FROM laptop
    WHERE hd<ALL(SELECT hd FROM pc);
    ```

   {% gp 2-2 %}
   {% asset_img select_5.png %}
   {% asset_img select_5_result.png %}
   {% endgp %}

6. （集合查询）查询所有硬盘容量为500G的电脑型号及其类型。
    ```sql
    SELECT product.model,type
    FROM laptop, product
    WHERE product,model=laptop.model AND hd=500
    UNION
    SELECT product.model, type
    FROM pc, product
    WHERE product.model=pc.model AND hd=500;
    ```

   {% gp 2-2 %}
   {% asset_img select_6.png %}
   {% asset_img select_6_result.png %}
   {% endgp %}

### 视图操作
{% note info %}
在 MySQL 中，视图是一种虚拟表，它的数据并不存储在数据库中，而是通过查询语句动态生成。
{% endnote %}

1. （视图创建）建立笔记本电脑所有信息的视图。
	```sql
	CREATE VIEW laptop_info AS
		SELECT maker, speed, ram, hd, price
		FROM product, laptop
		WHERE 
			type IN ('laptop') 
			AND product.model=laptop.model;
	```

	{% gp 3-3 %}
	{% asset_img create_view_1.png %}
	{% asset_img create_view_2.png %}
	{% asset_img create_view_3.png %}
	{% endgp %}

2. （视图更新）将制造商C的所有笔记本价格上调200元。
	```sql
	SET SQL_SAFE_UPDATES=0;
	UPDATE laptop_info
	SET price=price+200
	WHERE maker='C';
	```
 
	{% gp 2-2 %}
	{% asset_img update_view_1.png %}
	{% asset_img update_view_2.png %}
	{% endgp %}

3. （视图查询）在笔记本中找出屏幕大小为17寸的所有产品信息。
	```sql
	SELECT * FROM laptop_info
	WHERE screen=17;
	```
 
	{% gp 2-2 %}
	{% asset_img select_view_1.png %}
	{% asset_img select_view_2.png %}
	{% endgp %}

	{% note warning %}
    若screen的数据类型为double或float，类似于数据查询，也会导致查询失败！
	{% endnote %}

4. （视图删除）删除笔记本电脑信息视图。
	```sql
	DROP VIEW laptop_info;
	```
 
	{% gp 2-2 %}
	{% asset_img drop_view_1.png %}
	{% asset_img drop_view_2.png %}
	{% endgp %}

## 遇到的问题和解决方案
1. 使用update命令更新数据时出现1175错误。提示信息如下：
	```
	Error Code: 1175. You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column To disable safe mode, toggle the option in Preferences -> SQL Editor and reconnect.
	```

	MySQL有个叫`SQL_SAFE_UPDATES`的变量，为了数据库更新操作的安全性，此值默认为`1`，所以才会出现更新失败的情况。所以，出现1175错误的时候，可以先设置`SQL_SAFE_UPDATES`的值为`0`，然后再执行更新。对于上例来说，直接在前面加上一句`SET SQL_SAFE_UPDATES = 0; `即可解决问题。
 
2. 在MySQL中，`CREATE DATABASE`和`CREATE SCHEMA`的区别是什么？

	{% blockquote MySQL官方英文文档 https://dev.mysql.com/doc/refman/5.6/en/create-database.html %}
    “CREATE DATABASE creates a database with the given name. To use this statement, you need the CREATE privilege for the database. CREATE SCHEMA is a synonym for CREATE DATABASE.” 
	{% endblockquote %}

	`CREATE DATABASE`根据给定的名称创建数据库，你需要拥有数据库的`CREATE`权限来使用这个语句。`CREATE SCHEMA`是`CREATE DATABASE`的一个代名词。由此可见，在MySQL的语法操作中（MySQL5.0.2之后），可以使用`CREATE DATABASE`和`CREATE SCHEMA`来创建数据库，两者在功能上是一致的。在使用MySQL官方的MySQL管理工具MySQL Workbench 5.2.47创建数据库时，使用的是`CREATE SCHEMA`来创建数据库的。而这和MS SQL中的`SCHEMA`有很大差别。

3. 在MySQL修改基本表中列的数据类型时，使用的命令是`CHANGE COLUMN`。

