---
layout:     post
title:      "MongoDB基本篇"
date:       2018-12-08 12:30:00 +0800
author:     "Sky丶Memory"
header-img: "img/2018-12-08-01-bg.jpeg"
tags:
    - Storage
    - MongoDB
---
MongoDB是一款开源的、分布式文档型数据库产品，其提供了无结构、高性能、高可用、自动伸缩的特性吸引了不少开发者的青睐，公司里面不少项目中都有它的影子，所以打算学习学习。

MongoDB Docker仓库地址[bitnami/mongodb](https://hub.docker.com/r/bitnami/mongodb/)，本文中所有命令都在mongo shell环境下执行



#### 基本概念

| MongoDB | 关系型数据库 |
| :------: | :------: |
| db | database |
| collection | table |
| document | row |

上述表中的概念并不是严格对等，在MongoDB中不需要事先创建db或collection，插入文档时不存在对应的db或collection会自动创建

一条数据记录在MongoDB中定义为一个文档，内部以BSON作为其存储格式，BSON基本格式与JSON类似：
![](/img/2018-12-08-01-01.jpg)

切换到test数据库:

`use test `

查看数据库中的集合:

` show collections `

查看当前MongoDB实例中存在的数据库:

` show databases `

当前使用的是哪个数据库：

` db.getName() `

mongo shell中查看对象支持的方法，都可以通过名称.help()查看:

` db.help() `




#### CRUD

##### 插入

MongoDB插入支持单条、批量插入，基本使用方式:
```shell
> use test
> db.inventory.insertOne({ item: "canvas", qty: 100, tags: ["cotton"], size: { h: 28, w: 35.5, uom: "cm" } })
{
	"acknowledged" : true,
	"insertedId" : ObjectId("5c0a382808d4278148c007e1")
}
> db.inventory.insertMany([
...    { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
...    { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
...    { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
... ])
{
	"acknowledged" : true,
	"insertedIds" : [
		ObjectId("5c0a382d08d4278148c007e2"),
		ObjectId("5c0a382d08d4278148c007e3"),
		ObjectId("5c0a382d08d4278148c007e4")
	]
}
```
MongoDB默认会自动为每条文档生成一个ObjectID对象作为其唯一标识

##### 查询

测试数据集:
```shell
db.inventory.insertMany([
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "A" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" }
]);
```
查询所有文档:

```shell
db.inventory.find()
等价于SQL:
SELECT * FROM inventory
```
等值查询:
```shell
db.inventory.find( { status: "D" } )
等价SQL:
SELECT * FROM inventory WHERE status = "D"
```

MongoDB查询选择器支持如下比较操作符:

| Name | Description |
| :------: | :------: |
| $eq | Matches values that are equal to a specified value. |
| $ne | Matches all values that are not equal to a specified value.|
| $gt | Matches values that are greater than a specified value. |
| $gte | Matches values that are greater than or equal to a specified value. |
| $lt | Matches values that are less than a specified value. |
| $lte | Matches values that are less than or equal to a specified value. |
| $in | Matches any of the values specified in an array. |
| $nin | Matches none of the values specified in an array. |

比较操作符的语法格式为:

```{ <field1>: { <operator1>: <value1> }, ... } ```

比如查询qty大于等于50的文档:

```db.inventory.find({qty: {$gte: 50}})```

其他比较操作符与此类似，不做过多叙述

MongoDB支持四个逻辑运算符：

| Name | Description |
| :------: | :------: |
| $and | Joins query clauses with a logical AND returns all documents that match the conditions of both clauses. |
| $not | Inverts the effect of a query expression and returns documents that do not match the query expression. |
| $nor | Joins query clauses with a logical NOR returns all documents that fail to match both clauses. |
| $or | Joins query clauses with a logical OR returns all documents that match the conditions of either clause. |

一些使用例子:
```shell
db.inventory.find( { status: "A", qty: { $lt: 30 } } )
等价SQL:
SELECT * FROM inventory WHERE status = "A" AND qty < 30

db.inventory.find( { $or: [ { status: "A" }, { qty: { $lt: 30 } } ] } )
等价SQL:
SELECT * FROM inventory WHERE status = "A" OR qty < 30

db.inventory.find({qty: {$not: {$gt: 40}}})
等价SQL:
SELECT * FROM inventory WHERE NOT qty > 40

db.inventory.find({$nor: [{qty: 25}, {status: "D"}, {"item": "note"}]})
等价SQL:
SELECT * FROM inventory WHERE NOT qty = 25 and not status = "D" and not item = "note"

```

find函数支持两个参数，上面查询例子中仅使用了query参数，其第二个参数为projection，主要作用用于明确指定返回的字段，从而降低网络传输，基本语法:

` { field1: <value>, field2: <value> ... } `

value取值为0、1、true、false、[Projection Operators](https://docs.mongodb.com/manual/reference/operator/projection/)，例如指定返回文档仅包含item字段(_id默认返回):

` db.inventory.find({}, {item: 1}) `

更多关于MongoDB查询说明，可参考官方文档提供的例子[Query Documents](https://docs.mongodb.com/manual/tutorial/query-documents/)


##### 更新

MongoDB同样地支持单条、批量更新，不过MongoDB需要特定的更新操作符来完成更新，这里罗列比较常用的一些更新操作符:

| Name | Description |
| :------: | :------: |
| $currentDate | Sets the value of a field to current date, either as a Date or a Timestamp. |
| $inc | Increments the value of the field by the specified amount. |
| $min | Only updates the field if the specified value is less than the existing field value. |
| $max | Only updates the field if the specified value is greater than the existing field value. |
| $mul | Multiplies the value of the field by the specified amount. |
| $rename | Renames a field. |
| $set | Sets the value of a field in a document. |
| $unset | Removes the specified field from a document. |

更新使用例子:

```shell
db.inventory.updateOne({}, {$inc: {qty: 1}})
等价SQL:
UPDATE inventory SET qty = qty + 1 LIMIT 1

db.inventory.updateMany({status: "A"}, {$set: {status: "H"}})
等价SQL:
UPDATE inventory SET status = "H" where status = "A"
```
上述例子中MongoDB的更新与其对应的SQL并不完全对等，这里只是简要说明更新操作符的语义。

在MongoDB中，由于schema-less特性，更新数据时可能会导致一些
DDL语义，比如新增字段；另外，MongoDB的DDL和DML是混在一起的

##### 删除

删除数据比较简单，两个简单例子:

```shell
db.inventory.deleteMany({ status : "A" })

db.inventory.deleteOne( { status: "D" } )
```

关于MongoDB的删除，有个地方需要注意:***删除记录不释放磁盘空间***，这很容易理解，为避免记录删除后的数据大规模挪动，原记录空间不删除，只标记“已删除”即可，以后还可以重复利用，可通过db.repairDatabase()来整理，但这个过程比较缓慢

#### 索引

MongoDB索引使用B树，创建索引基本语法：

`db.collection.createIndex(keys, options)`

keys参数为一个健值对对象，健为索引字段，值为索引字段类型，1为升序，-1为降序;options参数可用于指定索引名称、是否唯一、存储引擎等


在inventory集合中给qty加上索引:

` db.inventory.createIndex({qty: 1}, {name: "idx_inventory_qty"}) `

查看inventory集合中存在的索引:

` db.inventory.getIndexes() `

删除inventory集合中qty字段索引:

`db.inventory.dropIndex({qty: 1}) `

查看inventory集合中索引占用内存空间的大小，单位为字节:

`db.inventory.totalIndexSize()`

inventory集合中创建(qty, status)联合唯一索引:

`db.inventory.createIndex({qty: 1, status: -1}, {unique: true})`

MongoDB支持多种类型索引，比如稀疏索引、哈希索引、TTL索引、全文索引等，暂时还没有用到这么多，就不做过多介绍，更多的索引说明可参见官方文档[Indexes](https://docs.mongodb.com/manual/indexes/)rr