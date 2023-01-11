# mysql文件目录

## mysql数据目录

可以使用如下命令查看数据mysql的数据目录，mysql在启动时会到这个文件夹下加载数据，并将运行期间的数据写入到该目录下。

```sql
show variables like '%datadir%';
```

| Variable_name | Value           |
| ------------- | --------------- |
| datadir       | /var/lib/mysql/ |

## mysql命令目录

/usr/bin和/usr/sbin，这个两个目录下存有mysql可执行文件。

## mysql配置文件目录

/etc/mysql，这个目录下存放了mysql的一些配置文件。

/usr/share/mysql，这个目录下也存放了mysql的一些配置文件。

## mysql存储引擎与文件系统的关系

我们通过mysql存取数据都是先给到mysql的存储引擎，然后存储引擎将这些数据按照一定的格式存放到文件系统中，而在我们查询的时候则从文件系统中加载我们要的数据返回给我们。

## mysql中的默认数据库

- mysql：存储了一些系统级别的表，比如用户表等等
- information_schema：维护了mysql软件中，所有其他数据库的信息（一些元数据）。比如表、视图、索引、存储过程等等
- performance_schema：保存mysql运行中一些状态的信息，指标等等
- sys：用视图的方式将information_schema和performance_schema结合起来

## mysql数据库文件

每创建一个数据库都会在/var/lib/mysql这个目录下生成相应名称的文件夹。

- db.opt：5.7存储了当前数据库的字符集、比较规则等等数据。8.0没有该文件，字符集相关的信息存在表文件里

- xxx.frm：5.7存储了表结构信息（innodb和myisam都是）。8.0没有此文件，innodb表信息放到了.ibd文件中

- xxx.ibd：innodb引擎的数据默认放在这个文件（独立表空间）中

- xxx.MYD：myisam引擎表的数据存放在这个文件中

- xxx.MYI：myisam引擎表的索引存放在这个文件中

- xxx.sdi：5.7无此文件。8.0myisam表信息文件

  
