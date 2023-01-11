# mysql字符集

在mysql5.7中如果不设置字符集，则默认使用**latin1**字符集，如果在创建数据库或者数据表的时候没有手动指定字符集则会使用**latin1**，这会导致插入带有中文的记录时出现乱码或者报错。

在mysql8.0中则不会出现上述问题，因为它默认使用**utf-8mb4**。

## 查看mysql数据库当前字符集

```sql
show variables like '%character%';
```

| Variable_name                | Value                      |
| ---------------------------- | -------------------------- |
| character_set_client         | utf8mb4                    |
| character_set_connection     | utf8mb4                    |
| ***character_set_database*** | ***utf8mb4***              |
| character_set_filesystem     | binary                     |
| character_set_results        | utf8mb4                    |
| ***character_set_server***   | ***utf8mb4***              |
| character_set_system         | utf8                       |
| character_sets_dir           | /usr/share/mysql/charsets/ |

其中character_set_database与character_set_server影响了默认情况下数据库与表的字符集

## 怎么修改mysql的默认字符集

windows修改my.ini，linux修改my.cnf文件

```
[mysqld]
## 添加行，指定服务字符集为utf8
character_set_server=utf8
```

重启mysql服务，修改后，新创建的数据库字符集为utf8（旧的数据库字符集不变）。数据表的字符集默认跟随数据库。数据库的字符集默认跟随服务