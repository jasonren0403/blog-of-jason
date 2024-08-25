---
title: Cypher 学习笔记
date: 2024-08-25 11:04:21
tags:
  - Cypher
  - Neo4j
categories:
  - ["图数据库", "Neo4j"]
---

由于学校实验室项目的需求，需要简单了解一下图数据库的使用，根据之前留下的基础，选择了 Neo4j 的 Cypher 语言。本篇文章整理了一些常见的 Cypher 语法。

<!-- more -->

## 简介

Cypher是Neo4j用于查询图形数据的语言，类似于SQL。Cypher关键字不区分大小写，但是属性值，标签，关系类型和变量是区分大小写的。

{% note info %}
以下介绍的语法均适用于 Neo4j 版本 4。
{% endnote %}

## CREATE 命令

节点及关系一般使用 `CREATE` 命令创建。

```cypher
CREATE (<node-name>:<label-name>)
```

* 创建带属性节点
    ```cypher
    CREATE (
        <node-name>:<label-name> {<property-name>: <property-value>, ...}
    )
    ```
* 创建关系

    {% codeblock lang:cypher mark:3 %}
    match(p1:<type1>), (p2:<type1>)
    where p1.<property-name> = <property-value> and p2.<property-name> = <property-value>
    create (p1)-[r:<relationship-type>]->(p2)
    return r
    {% endcodeblock %}

* 创建或更新：`MERGE`
    {% codeblock lang:cypher mark:1-2 %}
    MATCH (person:Person)          # 先查询，可以创建新变量
    MERGE (city:City { name: person.bornIn })           # 合并的变量，变量可能会已经存在
    ON CREATE SET keanu.created = timestamp()   # 如果是创建了变量          
    ON MATCH SET keanu.lastSeen = timestamp()   # 如果是更新了操作
    RETURN person.name, person.bornIn, city
    {% endcodeblock %}

## MATCH 命令

### 从数据库获取有关节点、关系和属性的数据

```cypher 
// 节点
MATCH(<node-name>:<label-name>)
// 相邻节点（用 `--` 标志）
MATCH(<node-name>:<label-name>)--(<node-name>:<label-name>)
// 方向 DIRECTED `<--` / `-->`
MATCH(NODE1)-->(NODE2)
MATCH(NODE1)<--(NODE2)
// 关系（边）
MATCH p=()-[<relationship-name>:<relationship-type>]->()
MATCH (NODE1)<-[r1]-(NODE2)-[r2]->(NODE2)
MATCH ()-[:label*3]-() // 3 hops=*3
```

### 只返回感兴趣的结果

`match (p:Person) return p` 返回所有节点，`match (p:Person) return p.name` 返回所有节点的 `name` 属性

```cypher
RETURN 
   <node-name>.<property1-name>,
   ........
   <node-name>.<propertyn-name>
```

{% note info %}
例如：选出 movie 数据库中所有与 Oliver Stone 有关的电影

```cypher
MATCH (:Person { name: 'Oliver Stone' })-->(movie)

RETURN movie.title
```
{% endnote %}

### WHERE 子句

* 读取节点
  ```cypher
  match (n)  // <--
  where id(n)=0
  return n
  ```
* 读取边
  ```cypher
  match ()-[r]->()  // <--
  where id(r)=0
  return r
  ```
* 批量读取
  ```cypher
  match (n)  
  where id(n) in [0,1,2] // <--
  return n
  ```
* 布尔运算符
	- AND、OR、XOR、NOT
	- 也可以用label、属性等过滤
* 包含属性判断
	- `n.name is not null` 判断n节点是否有name属性
	- 字符串：`STARTS WITH`、`ENDS WITH`、`CONTAINS`
* 孤立节点
	- `match (n) where not (n)--() return n` 找到没有关系的节点
	- `match (n) where not (n)-[]-() return n` 找到没有关系的节点
	- `match (n) where not (n)-[:RELATIONSHIP]-() return n` 找到没有RELATIONSHIP关系的节点

### 匹配一种或多种类型的关系

在表示关系的方括号中用 `|` 分隔

```cypher 
MATCH (wallstreet { title: 'Wall Street' })<-[:ACTED_IN|DIRECTED]-(person)
RETURN person.name
```

## DELETE 命令

```cypher 
// 删除节点
match (p: Person) 
where p.name="刘备"
delete p
 
// 删除关系
MATCH (p:Person)-[r:MY_GENERAL]-(g:Person) 
DELETE r
 
// 删除两个节点和一个关系
MATCH (cc: CreditCard)-[rel]-(c:Customer) 
DELETE cc,c,rel

// 删除数据库中所有的节点和其上绑定的关系
MATCH (n)
DETACH DELETE n
```

## 管道：WITH
聚合结果使用 `WITH` 子句进行过滤和排序：`ORDER BY`、`count(*)`、`limit`、`DISTINCT` 等。

{% note info 常见用途 %}
* 统计总节点数 `match (n) return count(n)`
* 统计总关系数 `match ()-->() return count(*)`
{% endnote %}

## SET 命令

### 设置属性
{% codeblock lang:cypher mark:2 %}
MATCH (n { name: 'Andy' })
SET n.surname = 'Taylor'
RETURN n.name, n.surname
{% endcodeblock %}

### 设置标签
{% codeblock lang:cypher mark:2-3 %}
MATCH (n { name: 'Stefan' })
SET n:German        # 设置标签
SET n:Swedish:Bossman   # 设置多标签
RETURN n.name, labels(n) AS labels
{% endcodeblock %}

### 节点或关系对象赋值

{% codeblock lang:cypher mark:2-6 %}
MATCH (at { name: 'Andy' }),(pn { name: 'Peter' })
SET at = pn          #会删除原有属性
SET p = { name: 'Peter Smith', position: 'Entrepreneur' }   #会删除原有属性
SET p = { }     #会删除原有属性
SET n = $props   #会删除原有属性
SET p += { age: 38, hungry: TRUE , position: 'Entrepreneur' }   #不会删除原有属性
RETURN at.name, at.age, at.hungry, pn.name, pn.age
{% endcodeblock %}

## 图算法

### 深度优先 bfs

{% codeblock lang:cypher mark:1 %}
MATCH (c1:City {name: "London"})-[edge_list:ROAD_TO *bfs..10]-(c2:City {name: "Paris"})
RETURN *;
{% endcodeblock %}

### 加权最短路径

{% codeblock lang:cypher mark:2 %} 
MATCH (c1:City {name: "London"})-[
    edge_list:ROAD_TO *wShortest 10 (e, n | e.weight) total_weight
  ]-(c2:City {name: "Paris"})
RETURN *;
{% endcodeblock %}

### 调用图算法

```cypher
CALL <algorithm_name> YIELD *
```

- 如 `graph_analyzer.analyze()`、`wcc.get_components(nodes/* collect(n) */, edges/* collect(e) */)`、`nxalg.pagerank()`

## 参考资料
* https://neo4j.com/docs/api/java-driver/current/org.neo4j.driver/org/neo4j/driver/package-summary.html
* https://neo4j.com/docs/cypher-cheat-sheet/current/
