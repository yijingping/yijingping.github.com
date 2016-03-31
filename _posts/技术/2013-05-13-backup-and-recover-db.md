---
layout: post
title: 简单的备份和恢复数据库表
category: 技术
tags: mysql
---

在执行数据库操作之前，一定要先做备份。

#. 备份数据库表:

```
mysqldump -uuser -ppassword -hhost database_name table_name > table_name.sql
```

如果提示:

```
mysqldump: Got error: 1044: Access denied for user 'user'@'host' to database 'database_name' when doing LOCK TABLES
```

则增加参数```--single-transaction```

#. 恢复数据库表:

```
mysql -uuser -ppassword -hhost database_name < table_name.sql
```

	
