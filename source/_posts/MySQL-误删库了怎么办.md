---
title: MySQL 误删库了怎么办
date: 2021-01-23 20:00:26
tags:
---
 > MySQL 由于没加条件更新了全表数据，或者误删库后只能跑路吗？显然不是的，本文简单谈谈MySQL 的备份、恢复策略。



#### 一般情况下MySQL 的备份分为以下几种：

- 逻辑备份

- 物理备份

- lvm 快照备份

- 主从备份


 逻辑备份就是 用mysqldump 工具定期导出sql，这种方式的优点是灵活、文本储存，有许多参数可供选择并且可直接查看数据信息。缺点是备份时间较长，备份时占用服务器资源较大，因此推荐在系统不繁忙时备份。


物理备份是直接打包 数据库的 data 目录 文件，优点是快，缺点是备份的数据易损坏，备份时需要给加锁，防止新数据写入。


lvm 是 linux 直接复制磁盘快照的方式备份 MySQL。



主从备份是通过 设置 MySQL 主从延迟时间的方式达到备份的目的，在主库误操作之后，只要误操作还没同步到从库，就可以通过 stop slave 的方式，停止从库的复制，这时只需要把库切换为从库就达到恢复的目的。



有备份当然可以恢复，如果没有备份策略 可以通过 mysqlbinlog 来恢复，mysqlbinlog  需要 开启 binlog 参数

```ini
[mysqld]
# binlog 日志路径配置
log-bin = /etc/mysql/logs/mysql-bin.log
# binlog 有效期
expire-logs-days = 14
# 单文件最大大小
max-binlog-size = 500M 
server-id = 1 
```
 MySQL 的binlog 文件  可以通过以下命令来查看
```bash
mysqlbinlog --no-defaults  -v -v  --base64-output=DECODE-ROWS  mysql-bin.000001
```
mysql-bin.000001 为当前要查看的binlog文件，也可以添加 -d 参数查看指定的库的binlog 

binlog 文件中有 at  1089695  及日志的记录点，和时间 #210109 14:57:42 如下：

```bash
BEGIN
/*!*/;
# at 1089695
# at 1089904
#210109 14:57:42 server id 1  end_log_pos 1089935 CRC32 0x3d0263c0   Xid = 17531684
COMMIT/*!*/;
# at 1089935
#210109 14:57:42 server id 1  end_log_pos 1090000 CRC32 0x8be6bcd5   Anonymous_GTID  last_committed=1844  sequence_number=1845  rbr_only=no  original_committed_timestamp=0  immediate_commit_timestamp=0  transaction_length=0
# original_commit_timestamp=0 (1970-01-01 08:00:00.000000 CST)
# immediate_commit_timestamp=0 (1970-01-01 08:00:00.000000 CST)
/*!80001 SET @@session.original_commit_timestamp=0*//*!*/;
/*!80014 SET @@session.original_server_version=0*//*!*/;
/*!80014 SET @@session.immediate_server_version=0*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1090000
#210109 14:57:42 server id 1  end_log_pos 1090083 CRC32 0xb6986227   Query  thread_id=435893  exec_time=0  error_code=0
```
MySQL 可以通过 指定 --start-datetime    --stop-datetime  来恢复指定时间点的数据，或者 --start-position     --stop-position 恢复指定位置的数据。

一般运维所说 数据库可以恢复到 一周内任意一个时间点的数据，即数据库按周逻辑备份，和 binlog 结合的方式。

还有一些闪回工具，例如美团开源的 MyFlash 就是基于 binlog 的反向操作，即 insert 一条数据，反向操作就是 delete 。


另外 一定要做好MySQL 的安全管理，例如权限分配，普通登录用户不开删库的权限，MySQL 还有一个 sql_safe_updates 参数，可以有效限制不带 where 条件的 delete 和 update 语句是无法执行的，MySQL 一定要加这个参数来保证安全，如果开发中有需求要删除整张表，可以通过以下流程删除：

# 先备份表
```
create table table_bak like drop_table；
insert into table_bak select * from drop_table；
```
# 临时关闭 sql_safe 并删除
```
set session sql_safe_updates=0;
drop table drop_table;
set session sql_safe_updates=1;
```