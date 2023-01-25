# binlog写入机制

binlog是在mysql的server层面的日志，记录了mysql执行的DDL和DML操作，并不只有innodb存储引擎才会有（redolog只有innodb才有），它的写入机制和redolog方式基本一致。

在事务执行过程中，相关sql的执行除了写undolog、redolog之外，数据还会被写入binlog buffer中（这个binlog buffer是跟随线程的），在事务提交时进行刷盘，此处刷盘机制和redolog基本相同，它受sync_binlog控制：

默认是0，每次提交的时候都只是写到page cache中，什么时候持久化由系统决定

设置为1时，每次提交都会刷入page cache，并且调用sync ()进行磁盘同步

设置为N时，代表每提交N个事务则进行一次磁盘同步

## 二阶段提交

由于binlog和redolog都用做数据恢复，但是由于分开写入则会带来一些问题：

假如先写redolog，后写binlog，redolog写入成功，binlog写入失败并宕机，重启后，redolog进行数据恢复，提交的事务被持久化，但是由于binlog未写入成功，进行数据备份或数据恢复时，数据丢失

假日先写binlog，后谢redolog，binlog写入成功，redolog写入失败并宕机，重启后，edolog进行数据恢复，提交的数据丢失，但binlog中有成功的记录，如果有主从同步则主从数据不一致

所以mysql在事务提交写redolog和binlog时使用了两阶段提交来保证逻辑一致性（假设是双1配置，都是事务提交时进行磁盘同步）：

1. 对redolog进行刷盘，并设置为prepare
2. 对binlog进行刷盘
3. 将redo log从perpare修改为commit

在崩溃恢复时，如果redolog为commit则无脑进行数据重写并提交，如果redolog为prepare则对binlog进行判断，如果binlog写入成功，则进行数据重写并提交，如果binlog写入不成功，则进行数据回滚

