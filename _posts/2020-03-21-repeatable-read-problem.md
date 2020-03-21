---
layout: post
title:  "REPEATABLE READ Problem"
date:   2020-03-21 17:00:38 +0800
categories: [tech, mysql, innodb]
tag: [MySQL, InnoDB]
---
InnoDB 默认的事务隔离级别是 REPEATABLE READ，它能保证在同一个事务里：从第一次读之后，每次读到的数据都是一致的。这意味着如果在同一个事务里执行多次简单 SELECT 语句（简单指不加锁读），这些 SELECT 读到的数据都互相一致。对于加锁的查询，例如 `SELECT...FOR UPDATE` 则不然，一个例子：

有这样一张表：

{% highlight sql %}
CREATE TABLE `innodb_test` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键 ID',
  `state` int(10) unsigned NOT NULL COMMENT 'state',
  `update_by` varchar(50) NOT NULL DEFAULT '' COMMENT 'update by',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='InnoDB test'
{% endhighlight %}

表初始状态：

{% highlight sql %}
mysql> select * from innodb_test;
+----+-------+-----------+
| id | state | update_by |
+----+-------+-----------+
|  1 |     0 |           |
+----+-------+-----------+
1 row in set (0.00 sec)
{% endhighlight %}

现在打开两个连接，分别开启两个事务，并在连接 2 里执行：

{% highlight sql %}
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)
 
mysql> select * from innodb_test;
+----+-------+-----------+
| id | state | update_by |
+----+-------+-----------+
|  1 |     0 |           |
+----+-------+-----------+
1 row in set (0.00 sec)
{% endhighlight %}

切到连接 1，执行更新并提交事务：

{% highlight sql %}
mysql> update innodb_test set state = 1, update_by = '1' where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
 
mysql> select * from innodb_test;
+----+-------+-----------+
| id | state | update_by |
+----+-------+-----------+
|  1 |     1 | 1         |
+----+-------+-----------+
1 row in set (0.00 sec)
 
mysql> commit;
{% endhighlight %}

再切到连接 2，执行：

{% highlight sql %}
mysql> select * from innodb_test;
+----+-------+-----------+
| id | state | update_by |
+----+-------+-----------+
|  1 |     0 |           |
+----+-------+-----------+
1 row in set (0.00 sec)
{% endhighlight %}

发现这行数据没有变成连接 1 里更新的结果，而改为 `select * from innodb_test for update;` 则会变。

> This is the default isolation level for `InnoDB`. [Consistent reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read) within the same transaction read the [snapshot](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_snapshot) established by the first read. This means that if you issue several plain (nonlocking)  `SELECT ` statements within the same transaction, these `SELECT ` statements are consistent also with respect to each other.

从 MySQL 官方文档关于 REPEATABLE READ 的说明可以找到造成上面现象的原因，在事务里第一次读时会生成一个快照（连接 2 在开始事务后第一次 SELECT 时，连接 1 的事务还没有提交），后续的简单 SELECT 都会去读这个快照，所以就算连接 1 的事务已经提交了，连接 2 第二次 SELECT 时读到的还是快照里的旧数据。`SELECT...FOR UPDATE` 时则会强制读取最新已提交的数据。