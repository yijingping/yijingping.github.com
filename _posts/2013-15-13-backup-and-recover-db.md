如何备份和恢复数据库
========================================

#. MySQL数据库表的备份
------------------------------------------
在执行数据库操作之前，一定要先做备份。

备份数据库表:
```
mysqldump -uuser -ppassword -hhost database_name table_name > table_name.sql
```

恢复数据库表:
```
mysql -uuser -ppassword -hhost database_name < table_name.sql
```

	
