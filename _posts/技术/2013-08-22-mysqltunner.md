---
layout: post
title: MySQL 数据库性能优化 —— 使用 MySQLTuner（4）
category: 技术
tags: mysql
---

[MySQLTunner] 是针对MySQL性能分析的一个perl脚本。它采集数据库的信息，生成分析报告，并给出一些优化建议。

[MySQLTunner]: https://github.com/major/MySQLTuner-perl 

# 安装和使用

    wget mysqltuner.pl
    perl mysqltuner.pl

接着提示输入用户名密码。

过一会后，会生成类似下面的分析报告:

     >>  MySQLTuner 1.2.0 - Major Hayden <major@mhtx.net>
     >>  Bug reports, feature requests, and downloads at http://mysqltuner.com/
     >>  Run with '--help' for additional options and output filtering
    Please enter your MySQL administrative login: root
    Please enter your MySQL administrative password: 
    
    -------- General Statistics --------------------------------------------------
    [--] Skipped version check for MySQLTuner script
    [OK] Currently running supported MySQL version 5.1.57-1.3-log
    [OK] Operating on 64-bit architecture
    
    -------- Storage Engine Statistics -------------------------------------------
    [--] Status: +Archive -BDB -Federated +InnoDB -ISAM -NDBCluster 
    [--] Data in MyISAM tables: 5G (Tables: 529)
    [--] Data in InnoDB tables: 53G (Tables: 223)
    [--] Data in MEMORY tables: 496K (Tables: 5)
    [!!] Total fragmented tables: 280
    
    
    -------- Security Recommendations  -------------------------------------------
    [OK] All database users have passwords assigned
    
    -------- Performance Metrics -------------------------------------------------
    [--] Up for: 32d 10h 18m 26s (7B q [2K qps], 380M conn, TX: 3924B, RX: 821B)
    [--] Reads / Writes: 78% / 22%
    [--] Total buffers: 4.5G global + 2.6M per thread (3000 max threads)
    [OK] Maximum possible memory usage: 12.2G (62% of installed RAM)
    [OK] Slow queries: 0% (384K/7B)
    [OK] Highest usage of available connections: 37% (1119/3000)
    [OK] Key buffer size / total MyISAM indexes: 16.0M/1.6G
    [OK] Key buffer hit rate: 99.2% (4B cached / 30M reads)
    [!!] Query cache efficiency: 19.3% (942M cached / 4B selects) 
    [!!] Query cache prunes per day: 3122939
    [OK] Sorts requiring temporary tables: 0% (496 temp sorts / 149M sorts)
    [!!] Temporary tables created on disk: 48% (216M on disk / 448M total)
    [OK] Thread cache hit rate: 99% (2M created / 380M connections)
    [!!] Table cache hit rate: 0% (440 open / 12M opened) 
    [OK] Open file limit used: 0% (17/15K)
    [OK] Table locks acquired immediately: 99% (3B immediate / 3B locks)
    [!!] InnoDB data size / buffer pool: 53.5G/4.0G
    
    -------- Recommendations -----------------------------------------------------
    General recommendations:
        Run OPTIMIZE TABLE to defragment tables for better performance
        Increasing the query_cache size over 128M may reduce performance
        When making adjustments, make tmp_table_size/max_heap_table_size equal
        Reduce your SELECT DISTINCT queries without LIMIT clauses
        Increase table_cache gradually to avoid file descriptor limits
    Variables to adjust:
        query_cache_limit (> 2M, or use smaller result sets)
        query_cache_size (> 256M) [see warning above]
        tmp_table_size (> 16M)
        max_heap_table_size (> 16M)
        table_cache (> 256)
        innodb_buffer_pool_size (>= 53G)

# 分析和优化

上面打“!!”表示需要注意的地方，最后还给出了优化建议。下面逐一分析

* __OPTIMIZE TABLE__ 
    
    `[!!] Total fragmented tables: 280` 表示有280张表有碎片，需要整理。
    
    使用
    
        SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA NOT IN 
        ('information_schema','mysql') AND Data_free > 0 AND NOT ENGINE='MEMORY'; 
    
    语句找出所有要优化的表，然后执行:
    
        OPTIMIZE TABLE table_name
    
    __注意__： OPTIMIZE TABLE 会锁表，应该在数据库空闲的时候操作
    
    如果想更深入的了解OPTIMIZE TABLE的作用和原理，阅读下面两篇文章：
    
    + [OPTIMIZE TABLE Syntax](http://dev.mysql.com/doc/refman/5.1/en/optimize-table.html)
    + [实例说明optimize table在优化mysql时很重要](http://blog.51yip.com/mysql/1222.html)

* __Use Slave__

    `[--] Reads / Writes: 78% / 22%` 本数据库作为主库，reads太高，检查程序，改读从库

* __Query Cache__

    查看配置

    my.cnf

        # * Query Cache Configuration
        #
        query_cache_limit       = 2M
        query_cache_min_res_unit = 2k
        query_cache_size    = 256M

    mysql

        mysql> show global status like '%Qcache%';
        +-------------------------+------------+
        | Variable_name           | Value      |
        +-------------------------+------------+
        | Qcache_free_blocks      | 64154      |
        | Qcache_free_memory      | 102580512  |
        | Qcache_hits             | 945695317  |
        | Qcache_inserts          | 1648773540 |
        | Qcache_lowmem_prunes    | 101479085  |
        | Qcache_not_cached       | 2286862149 |
        | Qcache_queries_in_cache | 121884     |
        | Qcache_total_blocks     | 308314     |
        +-------------------------+------------+

    可以计算出：

    `Qcache_inserts/(Qcache_inserts+Qcache_not_cached) = 41.9%` , 说明差不多一半sql都进入了缓存，缓存率挺高。

    `usage_rate =  (query_cache_size-Qcache_free_memory)/query_cache_size = 40%` , 有60%的缓存空间都没有利用，可以减半

    `hit_rate= Qcache_hits/(Qcache_hits+Qcache_inserts+Qcache_not_cached) = 19.3%` , 缓存命中率偏低

    `effectiveness = Qcache_hits/Qcache_inserts = 57%` , 对于已缓存的内容，重复利用率是57%

    `Qcache_prunes/Qcache_inserts = 6.1%` , 6.1%的缓存是没有用的 

    __结论：__ Query cache 太大，但命中率低，效果不好，反而影响性能，应该减小一半(128M)。

    先在MySQL命令行中更改： 

        mysql> set global query_cache_size=134217728;

    再在my.cnf中更改，防止数据库重启配置丢失：

        query_cache_size = 128M

    上面的计算公式参考了下面两篇文章：

	+ [MySQL Query Cache 小结](http://isky000.com/database/mysql-query-cache-summary)
	+ [query cache size](http://www.dbtuna.com/article/46/query_cache_size_%7C_the_performance_impact_of_the_MySQL_Query_Cache)

* __Temporary Table__

    查看创建的临时表数量

        mysql> show global status like 'created_tmp%';
        +-------------------------+-----------+
        | Variable_name           | Value     |
        +-------------------------+-----------+
        | Created_tmp_disk_tables | 216784465 |
        | Created_tmp_files       | 1227      |
        | Created_tmp_tables      | 233126878 |
        +-------------------------+-----------+
        3 rows in set (0.00 sec)

    查看创建的临时表大小限制

        mysql> show variables where variable_name in ('tmp_table_size', 'max_heap_table_size');
        +---------------------+----------+
        | Variable_name       | Value    |
        +---------------------+----------+
        | max_heap_table_size | 16777216 |
        | tmp_table_size      | 16777216 |
        +---------------------+----------+
        2 rows in set (0.00 sec)

    可以看出：

	`created_tmp_disk_tables / created_tmp_tables`  接近100%, 实际中应该是< 25%比较合适。

    说明 `max_heap_table_size = 16M` 太小了,大部分临时表都创建到硬盘而非内存中了。

    碰到这个问题的时候，首先应该排查代码的问题，如果可以优化查询，先优化查询。其次，才是将 `max_heap_table_size`、`tmp_table_size`的值设大一点(多大合适呢，应该去tmpdir下看临时表的大小或者逐渐增大，直到比例恢复到正常)。我设置成了256M。
    
    先在mysql命令行中更改，然后查看数据库性能状态变化，改到一个合适的值后，再写入到my.cnf文件，防止数据库重启丢失配置。

        mysql> set @@max_heap_table_size=268435456;
        mysql> set @@tmp_table_size=268435456;

    想了解哪些SQL语句会生成临时表，参考mysql官网说明：

    + [How MySQL Uses Internal Temporary Tables](http://dev.mysql.com/doc/refman/5.1/en/internal-temporary-tables.html)

* __Table Cache__

    查看table_cache的大小

    my.cnf
        
        max_connections=3000 
        table_cache=256

    调整 `table_cache = current_connections * every_connection_used_tables_count`，调整原因参考:
        
    + [How MySQL Opens and Closes Tables](http://dev.mysql.com/doc/refman/5.0/en/table-cache.html)

* __Innodb Buffer Pool Size__ 
    
    理论上 `innodb_buffer_pool_size ` 越大，innodb就越接近in-memory database，只要不会导致内存不够用就行。 
