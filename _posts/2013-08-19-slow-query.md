---
layout: post
title: MySQL慢查询分析
---


# 开启慢查询日志

* 方法1(推荐)：

    在 `/etc/mysql/my.conf` 中，将下面这段代码注释打开：
    
        #log_slow_queries   = /var/log/mysql/mysql-slow.log
        #long_query_time = 2                               
        #log-queries-not-using-indexes                     
    
    他们表示的意思是： 
    
    	`log-slow-queries` 指定慢查询日志的位置
    	`long_query_time` 指顶执行超过多久的sql会被记录下来， 这里是2秒。
    	`log-queries-not-using-indexes` 记录没有使用索引的query。
    
    __切记：改动完后，重启mysql__

* 方法2：
    
    如果正在运行的是线上库，不方便重启。MySQL从5.1及以后版本支持了 开启slow query in runtime。只需要执行：
    
        mysql> SET GLOBAL slow_query_log = 'ON';
    
    如果不生效，试一下：
    
        mysql> FLUSH LOGS;
    
    如果想要配置log的路径、超时时间等，参考下面这面这篇文章：
    
    + [linux下开启mysql慢查询，分析查询语句](http://blog.51yip.com/mysql/972.html)
    
    
# 日志分析工具  

可以直接使用more和tail查看log文件，获取实时的慢查询记录。

mysql自身也提供了mysqldumpslow辅助分析log文件。

使用 `man mysqldumpslow` 或者 `mysqldumpslow --help` 查看详细使用方法。

常用的查询如下:

    mysqldumpslow -s t -t 10 /var/log/mysql/mysql-slow.log
    得到返回累计查询时间最长的10个SQL。

    mysqldumpslow -s l -t 10 /var/log/mysql/mysql-slow.log
    得到累计锁表时间最长的10条SQL语句

下面是我从生产环境弄下来的数据：

    tiger@pear:/var/log/mysql$ mysqldumpslow -s t -t 2 -a mysql-slow.log
    
    Reading mysql slow query log from mysql-slow.log
    Count: 13  Time=51.96s (675s)  Lock=0.00s (0s)  Rows=9439298.0 (122710874), ssldeploy[ssldeploy]@[192.168.10.86]
      select city, name, house_price from community_info, community_price where community_info.id = community_price.community_id
    
    Count: 3  Time=3.33s (9s)  Lock=0.00s (0s)  Rows=1.0 (3), ssldeploy[ssldeploy]@[192.168.10.86]
      SELECT COUNT(*) FROM `member` WHERE (`member`.`name` LIKE '%13632694480%'  OR `member`.`email` LIKE '%13632694480%'  OR `member`.`mobile` LIKE '%13632694480%' )

我查询的是累计查询时间最长的两条记录。输出数据中各个字段的意思是：

* count: 查询次数
* Time: 平均查询时间（累计查询时间） 
* Lock: 平均锁表时间（累计所表时间） 
* Rows: 平均每次查询扫描的行数（累计扫描行数） 
* user@host: 查询用的账户和查询来源机器的ip。host有多个的时候显示的是_n_hosts  
* SQL: 慢查询语句。默认数字会变成N，字符串变成S，如果想显示原语句，加上参数`-a`

可以看出慢的原因，第一条数据是连表查询，并且表的记录数非常多; 第二条语句中使用了
LIKE查询，且使用了模糊匹配。之所以不会造成数据库出问题的原因是这两种查询次数(count)都很少，而且之来自与我们内部使用的管理后台(host)。


# 慢语句分析方法 

通常，慢查询问题会出现在四个方面:

* 代码中的查询语句不合理

    解决方法：[分析和优化MySQL查询语句](/2013/08/21/optimize-mysql-query.html)

* 数据库表设计不合理

    解决方法：[优化MySQL数据库表](/2013/08/21/optimize-mysql-table.html)

* 数据库性能有问题

    解决方法：[借助MySQLTuner优化MySQL](/2013/08/20/mysqltunner.html)

* 硬件问题

    解决方法：[MySQL服务器监控和优化](./)
