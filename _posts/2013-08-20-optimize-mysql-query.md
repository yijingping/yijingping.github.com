---
layout: post
title: MySQL 查询语句分析和优化（2）
---

通常我们使用mysql slow query log获取到慢查询语句后，需要对这些语句进行分析和优化;
或者我们写SQL的时候，不确定该语句的执行效率，也需要进行分析。

一般我们会用到 [explain] 和 [profiling] 这两个命令。 

# 使用explain分析查询

explain的用法很简单，只要在mysql中执行:

    mysql> explain SQL;

explain输出的各个字段的意思在下面这两篇文章中说的十分详细：

* [EXPLAIN Output Format](http://dev.mysql.com/doc/refman/4.1/en/explain-output.html)
* [Mysql Explain 详解](http://www.cnitblog.com/aliyiyi08/archive/2008/09/09/48878.html)

这里用这个命令来分析2条语句的优化:

    mysql> explain select id from community_info where `name` = '船营' AND `city` = '吉林' AND `status` = 0 ORDER BY accuracy desc LIMIT 0,1\G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: community_info
             type: ref
    possible_keys: name_index,city,index_status
              key: name_index
          key_len: 50
              ref: const
             rows: 1
            Extra: Using where; Using filesort
    1 row in set (0.00 sec)



再使用 `desc community_info` 或者 `show create table community_info` 查看表的基本信息。

    mysql> desc community_info;
    +-------------------+---------------------+------+-----+-------------------+-----------------------------+
    | Field             | Type                | Null | Key | Default           | Extra                       |
    +-------------------+---------------------+------+-----+-------------------+-----------------------------+
    | id                | int(11)             | NO   | PRI | NULL              | auto_increment              |
    | city              | varchar(20)         | YES  | MUL | NULL              |                             |
    | area              | text                | YES  |     | NULL              |                             |
    | name              | varchar(32)         | NO   | MUL | NULL              |                             |
    | addr              | text                | YES  |     | NULL              |                             |
    | longitude         | int(10)             | YES  |     | NULL              |                             |
    | latitude          | int(10)             | YES  |     | NULL              |                             |
    | accuracy          | float               | YES  | MUL | NULL              |                             |
    | status            | int(3) unsigned     | YES  | MUL | 0                 |                             |
    ......
    +-------------------+---------------------+------+-----+-------------------+-----------------------------+
    70 rows in set (0.00 sec)

可以看出语句中用到的name、city、accuracy、status都建有索引，那为什么还有`using filesort`呢？

原因从`key：name_index`可以看出，实际上MySQL执行时只用了name字段的索引，其他的都没有用。

MySQL先从索引中刷选出name=“船营”的数据集，然后在数据集中对accuracy使用filesort排序。如果想去除filesort，应该建立联合索引idx(name, accuracy)。

如下所示：

    mysql> create index name_accuracy_index on community_info(name, accuracy);
    Query OK, 0 rows affected (5.08 sec)
    Records: 0  Duplicates: 0  Warnings: 0
    
    mysql> explain select id from community_info where `name` = '船营' AND `city` = '吉林' AND `status` = 0 ORDER BY accuracy desc \G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: community_info
             type: ref
    possible_keys: name_index,city,name_accuracy_index
              key: name_accuracy_index
          key_len: 98
              ref: const
             rows: 1
            Extra: Using where
    1 row in set (0.00 sec)

这个时候`key: name_accuracy_index`，MySQL只使用了联合索引，而filesort不存在了。

# 使用profiling分析查询

如果觉得explain的信息不够详细，可以使用profiling命令获取SQL执行每一步消耗的系统资源信息。

## 开启

    mysql> select @@profiling; # 查看是否开启profiling，0表示为开启，1表示开启
    +-------------+
    | @@profiling |
    +-------------+
    |           0 |
    +-------------+

    mysql> set profiling=1; # 开启

## 查看 

    mysql> show profiles\G;     # 获取所有profile语句
    *************************** 1. row ***************************
    Query_ID: 1
    Duration: 0.01953650
       Query: select @@profiling
    *************************** 2. row ***************************
    Query_ID: 2
    Duration: 0.00007250
       Query: SELECT COUNT(*) FROM `publish_rentmodel`
    *************************** 3. row ***************************
    Query_ID: 3
    Duration: 9.79044700
       Query: select city, name, house_price from community_info, community_price where community_info.id = community_price.community_id limit 1000
    3 rows in set (0.00 sec)
    
    mysql> show profile memory,cpu, block io for query 3;    # 获取id为3的SQL的执行的详细信息
    +--------------------------------+----------+----------+------------+--------------+---------------+
    | Status                         | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
    +--------------------------------+----------+----------+------------+--------------+---------------+
    | starting                       | 0.000036 | 0.000000 |   0.000000 |            0 |             0 |
    | Waiting for query cache lock   | 0.000006 | 0.000000 |   0.000000 |            0 |             0 |
    | checking query cache for query | 0.000068 | 0.000000 |   0.000000 |            0 |             0 |
    | checking permissions           | 0.000005 | 0.000000 |   0.000000 |            0 |             0 |
    | checking permissions           | 0.000006 | 0.000000 |   0.000000 |            0 |             0 |
    | Opening tables                 | 0.000306 | 0.000000 |   0.000000 |            0 |             0 |
    | System lock                    | 0.000011 | 0.000000 |   0.000000 |            0 |             0 |
    | Waiting for query cache lock   | 0.000034 | 0.000000 |   0.000000 |            0 |             0 |
    | init                           | 0.000030 | 0.000000 |   0.000000 |            0 |             0 |
    | optimizing                     | 0.000014 | 0.000000 |   0.000000 |            0 |             0 |
    | statistics                     | 0.000029 | 0.000000 |   0.000000 |            0 |             0 |
    | preparing                      | 0.000019 | 0.000000 |   0.000000 |            0 |             0 |
    | executing                      | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |
    | Sending data                   | 3.186532 | 0.096006 |   0.040003 |         9320 |            48 |
    | Waiting for query cache lock   | 0.000015 | 0.000000 |   0.000000 |            0 |             0 |
    | Sending data                   | 2.459872 | 0.112007 |   0.012000 |         8952 |            16 |
    | Waiting for query cache lock   | 0.000018 | 0.000000 |   0.000000 |            0 |             0 |
    | Sending data                   | 2.363912 | 0.056003 |   0.008001 |         8032 |            40 |
    | Waiting for query cache lock   | 0.000016 | 0.000000 |   0.000000 |            0 |             0 |
    | Sending data                   | 1.779400 | 0.080005 |   0.016001 |         6664 |           672 |
    | end                            | 0.000020 | 0.000000 |   0.000000 |            0 |             0 |
    | query end                      | 0.000008 | 0.000000 |   0.000000 |            0 |             0 |
    | closing tables                 | 0.000019 | 0.000000 |   0.000000 |            0 |             0 |
    | freeing items                  | 0.000013 | 0.000000 |   0.000000 |            0 |             0 |
    | Waiting for query cache lock   | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |
    | freeing items                  | 0.000034 | 0.000000 |   0.000000 |            0 |             0 |
    | Waiting for query cache lock   | 0.000003 | 0.000000 |   0.000000 |            0 |             0 |
    | freeing items                  | 0.000003 | 0.000000 |   0.000000 |            0 |             0 |
    | storing result in query cache  | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |
    | logging slow query             | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |
    | cleaning up                    | 0.000005 | 0.000000 |   0.000000 |            0 |             0 |
    +--------------------------------+----------+----------+------------+--------------+---------------+
    31 rows in set (0.00 sec)

从上面可以看出，这条语句的耗时主要在sendding data，原因就是返回的结果集过大，
可以在应用端用limit和offset拆分再合并。以减小单条语句的执行时间。


`show profile` 除了可以查看cpu、io、memory、执行时间外，还可以查看其他参数，完整用法参考:

* [SHOW PROFILE Syntax](http://dev.mysql.com/doc/refman/5.5/en/show-profile.html)

## 关闭

执行结束后，关闭profiling：

    mysql> set profiling=0;


# 参考文章：

* [mysql性能优化-慢查询分析、优化索引和配置](http://www.oicto.com/mysql-explain-show/)
* [mysql profile使用](http://hi.baidu.com/changzheng2008/item/d35559ec4caedf2b5a7cfb74)
