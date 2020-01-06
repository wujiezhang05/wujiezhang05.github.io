---
title: MYSQL ibdata1 和 innodb_undo_tablespaces
category: DB
---

> 记一次现网bug：客户服务器上磁盘告警 mysql 数据库分区的磁盘已使用92%。
> 调查分析后发现文件 ${mysql_path}/data/ibdata1 居然超过500G。占用了大量空间导致磁盘空间告警
> 且ibdata1在不断变大中。
>
> kill掉数据库中耗时长的事务之后，ibdata1不再持续增大，但依然占用很大磁盘空间。事后对这个问题研究了一番。
> 记录下来研究的内容。以供参考

## ibdata1 是什么

看下官方解释： https://dev.mysql.com/doc/refman/5.7/en/innodb-system-tablespace.html
```
The system tablespace is the storage area for the InnoDB data dictionary, the doublewrite buffer, the change buffer, and undo logs. It may also contain table and index data if tables are created in the system tablespace rather than file-per-table or general tablespaces.
```
系统表空间是存储 innodb 数据字典，双写缓冲区，变更缓冲区，撤销日志的地方。 如果是在系统表空间创建的table，那么ta的表数据，索引数据也会在这个地方.
```
The system tablespace can have one or more data files. By default, a single system tablespace data file, named ibdata1, is created in the data directory.
```
系统表空间有一个或多个数据文件，默认只有一个数据文件**ibdata1**. 在数据目录下

MYSQL提供了工具 innochecksum 可以查看 ibdata1文件的内容。

```shell
innochecksum -S ibdata1
```
输出为
```shell
File::ibdata1
================PAGE TYPE SUMMARY==============
#PAGE_COUNT	PAGE_TYPE
===============================================
      88	Index page
  104221	Undo log page
      12	Inode page
       0	Insert buffer free list page
    2808	Freshly allocated page
      14	Insert buffer bitmap
      98	System page
       1	Transaction system page
       8	File Space Header
       6	Extent descriptor page
       8	BLOB page
       0	Compressed BLOB page
       0	Other type of page
===============================================
Additional information:
Undo page type: 104171 insert, 50 update, 0 other
Undo page state: 0 active, 157 cached, 2 to_free, 0 to_purge, 0 prepared, 104062 other
```
可以看出 undo log page占了绝大部分。

## ibdata1中undo log为什么会有如此多呢 ?

undo log的作用是 回滚 以及支持 MMVC（mluti-version concurrency control）即多版本并发控制。

简单来说就是: 两个事务 一个在更新，一个在查询。 为了防止查询的事务读到脏数据或者更新一般的数据，更新的操作会在数据上排它锁，在更新事务未完成前对其他事务不可见。
这时候查询事务读到的数据 其实是数据的一个快照(通过undo读取某个版本)。MVVC就是同一份数据临时保留多版本的一种方式，进而实现并发控制

在这样的背景下，如果一个查询操作执行了很长的时间，那么数据库需要维护长时间的undo log.

通过下面的MYSQL 命令查找数据库的状态
```mysql
SHOW ENGINE INNODB STATUS/G;
```

```
---TRANSACTION 36E, ACTIVE 1256288 sec
MySQL thread id 42, OS thread handle 0x7f8baaccc700, query id 7900290 localhost root
show engine innodb status
Trx read view will not see trx with id >= 36F, sees < 36F
```
可以看到这条query 执行了14天还没有结束。需要在undo log中维护旧数据页以保障数据一致性视图。如果在这段时间数据库有大量写入任务，就意味着存储了大量的undo log


在问题发生时，我们也是通过
```
use information_schema;
select * from INNODB_TRX\G;
```
找到事务的 **trx_mysql_thread_id** 并
```
kill <trx_mysql_thread_id>
```
来杀掉长查询来停止ibdata1 不断增长的。

## 如何解决

1.  如何阻止ibdata1不断增长
  杀掉长时间未结束的事务,来阻止ibdata1的不断变大.
  但这并不能减小ibdata1的大小. **MYSQL innoDB 表空间从不收缩**(参考 [MYSQL ibdata1](http://bugs.mysql.com/bug.php?id=1341)),
  即便你删除数据, ibdata1中undo log页也只是标记为已删除稍微重用. 但不会释放这些空间给操作系统.

2. 如何回收表空间
  如上所述, MYSQL 表空间从不收缩,  除非用新的ibdata1启动数据库.  也就是说要mysqldump 做一个逻辑全备份, 然后停止MYSQL, 删除所有数据库, ib_logfile ibdata1等文件.  当你重新启动数据库时会产生一个新的ibdata1.  然后恢复逻辑备份.

3.  如何避免后续再发生此类事件吗
如果已经发生ibdata1 因为undo log过大的问题,那么除了删除数据文件重新启动没有更好的办法,

但是从MYSQL5.6 版本后 可以通过  创建外部undo log表空间 从而其从ibdata1中分离处来, 并设置相应参数来规定undo log的最大size等手段来达到在线清除undo 表空间

  - innodb_undo_directory 指定单独存放undo 表空间的目录
  - innodb_undo_tablespaces  指定单独存放undo 表空间的个数. 如果设置为3，则undo表空间为undo001、undo002、undo003
  - innodb_undo_logs 指定回滚段的个数（早期版本该参数名字是innodb_rollback_segments），默认128个。每个回滚段可同时支持1024个在线事务。这些回滚段会平均分布到各个undo表空间中。该变量可以动态调整，但是物理上的回滚段不会减少，只是会控制用到的回滚段的个数。
  - innodb_max_undo_log_size，undo表空间文件超过此值即标记为可收缩，默认1G，可在线修改；
  - innodb_purge_rseg_truncate_frequency,指定purge操作被唤起多少次之后才释放rollback segments。当undo表空间里面的rollback segments被释放时，undo表空间才会被truncate。由此可见，该参数越小，undo表空间被尝试truncate的频率越高。

--------------------------------------------
后记: 这个问题最后的解决办法 正是
1. 先杀掉耗时长的事务
2. 备份数据, 删除数据库文件
3. 增加innodb_undo_tablespaces等参数到数据库配置文件中. 重启数据库. 导入数据.
