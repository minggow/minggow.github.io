---
layout: post
title: MySQL 存储引擎
category: other
tags: [other]
excerpt:  MySQL 存储引擎
---

## MySQL 存储引擎是什么
MySQL不同于其他数据库只有一种存储引擎，具备一种特性:"Pluggable Storage Engine Architecture"，即可替换存储引擎架构。MySQL提供了多种
存储引擎，用户可以根据不同的需求为数据表选择不同的存储引擎，或者自己定义和扩展存储引擎。MySQL的工作其实分为语句分析层和存储引擎层，其中：
- 语句分析层：负责与客户端完成连接并事先分析语句的功能和内容
- 存储引擎层：负责接收语句分析层的分析结果，完成影响的数据输入和输出操作。
存储引擎就是如何存储数据、为何为存储的数据建立索引和如何更新、查询数据等技术的实现方法，因为在关系数据库中存储是以表的形式存储的，所以存储
引擎也可以称为表类型
![](/assets/images/2019/03/mysql_arch.jpg)

## MySQL 存储引擎分类
通过 ``show engines;`` 查看MySQL支持存储引擎的情况。
```
mysql>show engines;
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                                    | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                                         | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                                      | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                                         | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears)             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                                      | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Percona-XtraDB, Supports transactions, row-level locking, and foreign keys | YES          | YES  | YES        |
| ARCHIVE            | YES     | Archive storage engine                                                     | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables                  | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                             | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```

- Support: MySQL 是否支持该存储引擎
- Comment: 存储引擎的注释
- Transactions: 是否支持事务 
- XA: 是否支持分布式
- Savepoints: 是否支持save point的事务点操作

实际开发中经常使用的是**MyISAM**和**InnoDB**

## MyISAM存储引擎

MyISAM是MySQL中常见的存储引擎，曾经是MySQL的默认存储引擎。MyISAM是基于ISAM引擎发展起来的，增加了许多有用的扩展。
MyISAM的表存储成3个文件。文件的名字与表名相同。拓展名为frm、MYD、MYI。其实，frm文件存储表的结构；MYD文件存储数据，是MYData的缩写；MYI文件存储索引，是MYIndex的缩写。
基于MyISAM存储引擎的表支持3种不同的存储格式。包括静态型、动态型和压缩型。其中，静态型是MyISAM的默认存储格式，它的字段是固定长度的；
动态型包含变长字段，记录的长度不是固定的；压缩型需要用到myisampack工具，占用的磁盘空间较小。


### 索引结构
- 主索引结构
![主索引](/assets/images/2019/03/mysql_engine_myisam_primary.png)
- 辅助索引结构
![辅助索引](/assets/images/2019/03/mysql_engine_myisam_secondary.png)

## InnoDB存储引擎
InnoDB给MySQL的表提供了事务处理、回滚、崩溃修复能力和多版本并发控制的事务安全。在MySQL从3.23.34a开始包含InnnoDB。
它是MySQL上第一个提供外键约束的表引擎。而且InnoDB对事务处理的能力，也是其他存储引擎不能比拟的。靠后版本的MySQL的默认存储引擎就是InnoDB。
InnoDB存储引擎总支持AUTO_INCREMENT。自动增长列的值不能为空，并且值必须唯一。MySQL中规定自增列必须为主键。在插入值的时候，
如果自动增长列不输入值，则插入的值为自动增长后的值；如果输入的值为0或空（NULL），则插入的值也是自动增长后的值；如果插入某个确定的值，
且该值在前面没有出现过，就可以直接插入。
InnoDB还支持外键（FOREIGN KEY）。外键所在的表叫做子表，外键所依赖（REFERENCES）的表叫做父表。父表中被字表外键关联的字段必须为主键。
当删除、更新父表中的某条信息时，子表也必须有相应的改变，这是数据库的参照完整性规则。

### 索引结构
- 主索引结构
![主索引](/assets/images/2019/03/mysql_engine_innodb_primary.png)
- 辅助索引结构
![辅助索引](/assets/images/2019/03/mysql_engine_innodb_secondary.png)

## 比较总结
- 事务：InnoDB 是事务型的，可以使用 Commit 和 Rollback 语句。
- 并发：MyISAM 只支持表级锁，而 InnoDB 还支持行级锁。
- 外键：InnoDB 支持外键。
- 备份：InnoDB 支持在线热备份。
- 崩溃恢复：MyISAM 崩溃后发生损坏的概率比 InnoDB 高很多，而且恢复的速度也更慢。
- 其它特性：MyISAM 支持压缩表和空间数据索引

### 存储引擎选择
- 采用MyISAM
    - R/W > 100:1 且update相对较少
    - 并发不高
    - 表数据量小
    - 硬件资源有限
- 采用InooDB引擎
    - R/W比较少，频繁更新大字段
    - 表数据量超过1000万，并发高
    - 安全性和可用性要求高

![检索过程比较](/assets/images/2019/03/mysql_engine_index_process.png)


参考：
- [史上最简单MySQL教程详解（进阶篇）之存储引擎介绍及默认引擎设置](https://blog.csdn.net/m0_37888031/article/details/80664138)
- [Mysql存储引擎比较](https://blog.csdn.net/len9596/article/details/80206532)
- [MySQL存储引擎](https://blog.csdn.net/kang389110772/article/details/51233897)
- [四种mysql存储引擎](https://www.cnblogs.com/wcwen1990/p/6655416.html)
- [MySql两种存储引擎的区别](https://www.cnblogs.com/wangdake-qq/p/7358322.html)