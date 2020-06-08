---
layout:     post
title:      "MySQL分区表"
date:       2020-06-07 08:00:00 +0800
author:     "Sky丶Memory"
header-img: "img/2020/06/08-01-bg.jpg"
tags: 存储	

---

#### 背景

在日常的开发中，或多或少会遇上MySQL单表存储的数据量太大导致不得不分表的情况。水平分表是解决单表数据量过大的一种方式，而MySQL自身提供了分区表的功能也可以从一定程度上解决数据量过大的问题。

通常情况下，数据分区需要指定分区规则便于将数据存储在不同的物理文件中，分区的规则一般需要考虑两个要点：是否支持多列、是否支持索引。

#### 分区类型

目前MySQL支持RANGE、LIST、HASH、KEY四种分区，下面我们逐一介绍每种分区。

##### RANGE分区

使用语法：

```mysql
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY RANGE ( YEAR(separated) ) (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN (1996),
    PARTITION p2 VALUES LESS THAN (2001),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```

p3分区是个可选分区，如果没有指定，当插入大于等于2001年的数据会得到报错。

插入两行测试数据，查询时带上分区字段，验证分区裁剪是否有效。

````mysql
mysql> insert into employees(id, fname, lname, hired, separated, job_code, store_id) values(1, 'Sky', 'Memory', '2017-04-05', '2020-06-01', 10, 2), (2, 'Y', 'L', '2015-06-01', '2017-08-05', 20, 3);
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> explain select * from employees where separated between '2020-01-01' and '2020-12-31';
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | employees | p3         | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
````

从查询计划的执行结果来看，当查询时带上分区字段，MySQL仅会检索相关的分区数据。

既然MySQL内部提供了分区功能，那针对DATETIME、TIMESTAMP的分表方案如果用分区功能来实现，那是怎样的呢？

MySQL内部提供了一系列函数，如YEAR、TO_DAYS、TO_SECONDS、UNIX_TIMESTAMP可以帮助我们更好的针对时间相关字段进行分区。

DATETIME字段按月分表:

```mysql
CREATE TABLE my_range_datetime(
    id INT,
    hiredate DATETIME
) 
PARTITION BY RANGE (TO_DAYS(hiredate) ) (
    PARTITION p1 VALUES LESS THAN ( TO_DAYS('2020-01-01') ),
    PARTITION p2 VALUES LESS THAN ( TO_DAYS('2020-02-01') )
);
```

UNIX_TIMESTAMP字段按月分表：

```mysql
CREATE TABLE my_range_timestamp (
    id INT,
    hiredate TIMESTAMP
)
partition by range (unix_timestamp(hiredate))(
partition p1 values less than (unix_timestamp('2020-01-01 00:00:00')),
partition p2 values less than (unix_timestamp('2020-02-01 00:00:00'))
);
```

##### RANGE COLUMNS分区

RANGE COLUMNS分区可看成是RANGE分区的一种扩展，两者之间的差异主要体现在：

- RANGE COLUMNS不接受表达式，分区键可指定多个列名称。
- 列的类型不再受限于整型。

使用方式：

```mysql
CREATE TABLE rc4 (
    a INT,
    b INT,
    c INT
)
PARTITION BY RANGE COLUMNS(a,b,c) (
    PARTITION p0 VALUES LESS THAN (0,25,50),
    PARTITION p1 VALUES LESS THAN (10,20,100),
    PARTITION p2 VALUES LESS THAN (10,30,50)
    PARTITION p3 VALUES LESS THAN (MAXVALUE,MAXVALUE,MAXVALUE)
 );
```



##### LIST分区

LIST分区是枚举值列表的集合，分区语法定义为 PARTITION BY LIST(*`expr`*)，其中expr返回值需要是一个整型值。

```mysql
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LIST(store_id) (
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);
```

##### LIST COLUMNS分区

LIST COLUMNS分区是LIST分区的一种扩展，两者之间主要的差异点体现在：

- LIST COLUMNS支持多列、非整型类型

使用方式：

```mysql
CREATE TABLE customers_1 (
    first_name VARCHAR(25),
    last_name VARCHAR(25),
    street_1 VARCHAR(30),
    street_2 VARCHAR(30),
    city VARCHAR(15),
    renewal DATE
)
PARTITION BY LIST COLUMNS(city) (
    PARTITION pRegion_1 VALUES IN('Oskarshamn', 'Högsby', 'Mönsterås'),
    PARTITION pRegion_2 VALUES IN('Vimmerby', 'Hultsfred', 'Västervik'),
    PARTITION pRegion_3 VALUES IN('Nässjö', 'Eksjö', 'Vetlanda'),
    PARTITION pRegion_4 VALUES IN('Uppvidinge', 'Alvesta', 'Växjo')
);
```



##### HASH分区

语法定义为PARTITION BY HASH (*`expr`*)，其中expr返回值需要是一个整形值。

```mysql
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY HASH(store_id)
PARTITIONS 4;
```

PARTITIONS 4指定了分区数量，MySQL内部是根据公式 N = MOD(expr, NUM) 得出数据存放的分区编号，其中NUM含义为分区数量。

比如有如下建表语句：

```mysql
CREATE TABLE t1 (col1 INT, col2 CHAR(5), col3 DATE)
    PARTITION BY HASH( YEAR(col3) )
    PARTITIONS 4;
```

当插入一行记录，其col3值为2005-09-15时，该数据存放的分区编号计算方式为：

```mysql
MOD(YEAR('2005-09-15'), 4)
= MOD(2005, 4)
= 1
```

##### LINEAR HASH分区

LINEAR HASH分区跟常规的HASH分区类似，唯一的区别在于所采用的hash函数不同。

使用方式：

```mysql
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LINEAR HASH( YEAR(hired) )
PARTITIONS 4;
```

LINEAR HASH分区的优点在于，增加、删除、合并、拆分分区速度更快，比较适合处理TB级别的数据，缺点是，相对于HASH分区，它数据分布不均匀的概率更大。

##### KEY分区

KEY分区与HASH分区有些类似，主要存在两个区别点：

- HASH分区的hash函数由用户自定义，KEY分区的hash函数由MySQL内部定义。
- KEY分区支持多列。

使用方式：

```mysql
CREATE TABLE tk (
    col1 INT NOT NULL,
    col2 CHAR(5),
    col3 DATE
)
PARTITION BY  KEY (col1, col3)
PARTITIONS 3;
```

#### 分区管理

分区管理涉及四个操作：增加、删除、合并、拆分。

建表DDL：

```mysql
CREATE TABLE members (
    id INT,
    fname VARCHAR(25),
    lname VARCHAR(25),
    dob DATE
)
PARTITION BY RANGE( YEAR(dob) ) (
    PARTITION p0 VALUES LESS THAN (1980),
    PARTITION p1 VALUES LESS THAN (1990),
    PARTITION p2 VALUES LESS THAN (2000)
);
```

添加分区：

```mysql
ALTER TABLE members ADD PARTITION (PARTITION p3 VALUES LESS THAN (2010));
```

删除某个分区：

```mysql
ALTER TABLE members DROP PARTITION p2
```

拆分某个分区：

```mysql
ALTER TABLE members REORGANIZE PARTITION p0 INTO (
    PARTITION s0 VALUES LESS THAN (1960),
    PARTITION s1 VALUES LESS THAN (1970),
    PARTITION s2 VALUES LESS THAN (1980)
);
```

合并指定分区：

```mysql
ALTER TABLE members REORGANIZE PARTITION s0,s1 INTO (
    PARTITION p0 VALUES LESS THAN (1970)
);
```

REORGANIZE PARTITION通用语法：

```mysql
ALTER TABLE tbl_name
REORGANIZE PARTITION partition_list
INTO (partition_definitions);
```

#### 分区限制

分区表中，分区表达式使用的所有列必须是每个唯一索引的一部分。



