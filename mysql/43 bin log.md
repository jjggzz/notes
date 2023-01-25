# bin log

binlog是mysql中比较重要的日志，在开发有、运维当中经常会遇到。

bin log即binary log，二进制日志文件，也叫变更日志文件，他记录了数据库的所有DDL和DML等数据库更新事件的语句，但是不包含查询语句。

它主要用在数据恢复和数据复制的时候：

数据恢复：如果mysql意外停止，可以通过bin log来查看用户执行了哪些操作，对数据库服务器文件做了哪些修改，然后根据二进制日志文件中的记录来恢复数据库服务器。

数据复制：由于日志的延续性和时效性，可以通过它将数据复制给slave来达到主从复制的效果

## 查看配置

```sql
## 查看mysql中bin log相关的参数
show variables like '%log_bin%';
## 查看系统中所有的bin log文件
show binary logs ;
## 查看当前正在写入的bin log文件
show master status;
```

| variable_name                   | Desc                                                |
| ------------------------------- | --------------------------------------------------- |
| log_bin                         | 是否开启bin log功能                                 |
| log_bin_basename                | bin log文件夹                                       |
| log_bin_index                   | bin log文件索引                                     |
| log_bin_trust_function_creators | 是否信任函数，如果为OFF，则我们手动创建函数时会报错 |
| log_bin_use_v1_row_events       |                                                     |
| sql_log_bin                     | 是否记录DDL，DML这些修改语句                        |

我们可以对bin log做一些配置，只需修改mysql的配置文件即可（/etc/mysql/mysql.cnf）：

```
[mysqld]
# bin log文件存放的位置
log-bin=jgz-bin
# 指定bin log文件存活时间单位秒，超过则删除
# 默认情况下保存30天
binlog_exire_logs_seconds=600
# 指定bin log文件最大大小，超过则创建一个新的，mysql重启时，不管大小，都会创建一个新的
# 默认情况下是1G，但是这个大小并不严格，为了保证事务的完整性，可能会超出
max_binlog_size=100M
```

**注意：数据库文件最好不要与bin log文件放在一个磁盘上，一翻车就全没了**

## 查看bin log文件内容

由于bin log是二进制日志，无法像其他文本日志那样打开直接看，需要借助一些工具来查看：

**使用mysqlbinlog工具**，mysqlbinlog是一个查看mysql二进制日志的工具，可以把mysql上面的所有操作记录从日志里导出，这个工具默认的安装路径为：/usr/local/mysql/bin/mysqlbinlog

```shell
mysqlbinlog -v --base64-output=DECODE-ROWS "binlog文件路径"
```

**使用mysql命令**查看bin log文件内容

```sql
## 查看mysql-bin.000002这个bin log文件的内容
show binlog events in 'mysql-bin.000002';
```

## bin log存储格式

**statment**：每一条修改的sql都会记录到binlog中，优点是不需要记录每一行的变化（只记录sql），减少了binlog日志量

**row**：5.5.1开始支持的row level级别的复制，它不记录sql语句上下文信息，仅保存相关修改（记录哪条数据被改成什么了）。优点是会非常清楚的记录下每一行数据修改的细节。而且不会出现某些特定情况下的存储过程或者函数，以及触发器无法被正确复制的问题（默认row）

**mixed**：statment和row的混合

## 使用bin log进行数据恢复

现在有一个场景，不小心误操作删除了一些数据，现在想将这些被误删的数据找回来：

```sql
## 重新创建一个新的binlog文件，使后续操作不会影响到我们要操作的文件
flush logs;
```

做了上述操作之后，现在我们有两种恢复数据的方式，一种是使用position的方式，一种是使用时间的方式：

position方式

```sql
## 查看binlog内容
show binlog events in 'mysql-bin.000002';
## 找到要恢复的操作的开始pos和结束pos （这个一步很麻烦）
```

```shell
## start-position指定要恢复的开始位置
## stop-position指定要恢复的结束位置
## database指定要恢复的数据库
## 指定binlog文件路径
## 登陆到mysql
/usr/bin/mysqlbinlog --start-position=930 --stop-position=1025 --database=demo /var/lib/mysql/binlog/mysql-bin.000002 | /usr/bin/mysql -uroot -p123456 -v demo
```

使用时间的方式

```shell
## 查看binlog内容
mysqlbinlog -v --base64-output=DECODE-ROWS "binlog文件路径"
## 找到要恢复到操作开始的时间和结束的时间
```

```shell
/usr/bin/mysqlbinlog --start-datetime="2023-01-12 12:12:12" --stop-datetime="2023-01-12 12:12:13" --database=demo /var/lib/mysql/binlog/mysql-bin.000002 | /usr/bin/mysql -uroot -p123456 -v demo
```

## binlog 日志删除

```sql
## 删除2之前的binlog日志文件（2不会被删除）
purge master logs to 'mysql-bin.000002';
## 删除2022-01-12之前的binlog日志文件
purge master logs before '20220112';
## 将binlog日志全部清空（不要用）
reset master;
```

## 场景

一般来讲**binlog日志**可以和数据库的**全量备份**一起配合使用，完成数据库的**无损恢复**，但是如果遇到数据量很大、数据库和数据表很多的场景，用binlog进行数据恢复，是很有挑战性的，因为起止位置不容易管理。

