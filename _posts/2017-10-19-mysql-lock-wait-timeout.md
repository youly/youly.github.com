---
layout: post
title: 从线上lock wait timeout报错看mysql锁设计 
category: 设计
tag: [mysql, lock]
---

### 问题
最近监控邮件报告某个接口超时比较多，查看错误日志发现偶尔会有类似下面的记录：

    Exception occurs while doing  org.springframework.dao.CannotAcquireLockException:
    ### Error updating database.  Cause: java.sql.SQLException: Lock wait timeout exceeded; try restarting transaction

刚开始怀疑不是这个报错导致的，因为最近也没有上线，问了dba说数据库也没发现死锁异常，查看相关代码，事务包含的业务逻辑比较简单，看起来也不会出现比较耗时的操作。只是怀疑，于是关闭了某个可疑功能的开关，观察了几个小时，发现此类错误日志有所减少，但仍然有。想起前些天在线上跑官方账号粉丝数据，也是和这个功能相关，因此大概确定了问题：原本不大的表，因跑官方账号数据的任务短时间内插入了大量数据，虽然此表有建索引，但由于是一个账号有大量粉丝，执行更新的sql语句仍然要查找很多数据，导致这条sql执行时间长，又因为是在事务中，当前线程持有相关锁，导致其他线程无法获取相关锁而超时。

解决方法很简单，找dba优化下索引就行了。确实加完索引后，报错日志没有了，但有一个问题还没搞清楚，这里的相关锁是什么？

### 复现

在本机搭建mysql server环境，存储引擎默认 innodb，其他信息如下：

    mysql> select version();
    +------------+
    | version()  |
    +------------+
    | 5.6.24-log |
    +------------+
    
    mysql> show variables like 'innodb_locks_unsafe_for_binlog';
    +--------------------------------+-------+
    | Variable_name                  | Value |
    +--------------------------------+-------+
    | innodb_locks_unsafe_for_binlog | OFF   |
    +--------------------------------+-------+


假设使用如下表结构进行模拟：

    CREATE TABLE `tb` (
     `id` bigint(20) unsigned NOT NULL,
     `name` varchar(128) NOT NULL DEFAULT '',
     `tag` varchar(128) NOT NULL DEFAULT '',
     PRIMARY KEY (`id`),
     KEY `idx_name` (`name`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4

    -- 初始化数据
    insert into tb values (1,'a','t1');
    insert into tb values (2,'b', 't2');
    insert into tb values (3,'c', 't3');

**来看一下几种情况：**

##### scene1

| session1 | session2 |
| :------ | :------ |
| mysql> begin; | mysql> begin; |
| |mysql> update tb set tag = 't' where name = 'c'; |
||Query OK, 1 row affected (0.01 sec)|
|mysql> insert into tb values (4,'d', 't4');||
|ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction||

查看当前有哪些锁等待：

    mysql> select * from information_schema.innodb_lock_waits;
    +-------------------+-------------------+-----------------+------------------+
    | requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
    +-------------------+-------------------+-----------------+------------------+
    | 9406882           | 9406882:8712:4:1  | 9406881         | 9406881:8712:4:1 |
    +-------------------+-------------------+-----------------+------------------+

可以看出，事务9406882在等待事务9406881释放锁，锁信息如下：

    mysql> select * from information_schema.innodb_locks\G
    *************************** 1. row ***************************
        lock_id: 9406882:8712:4:1
    lock_trx_id: 9406882
      lock_mode: X
      lock_type: RECORD
     lock_table: `test`.`tb`
     lock_index: idx_name
     lock_space: 8712
      lock_page: 4
       lock_rec: 1
      lock_data: supremum pseudo-record
    *************************** 2. row ***************************
        lock_id: 9406881:8712:4:1
    lock_trx_id: 9406881
      lock_mode: X
      lock_type: RECORD
     lock_table: `test`.`tb`
     lock_index: idx_name
     lock_space: 8712
      lock_page: 4
       lock_rec: 1
      lock_data: supremum pseudo-record
    2 rows in set (0.00 sec)

##### scene2

| session1 | session2 |
| :------ | :------ |
| mysql> begin; | mysql> begin; |
| |mysql> update tb set tag = 't' where name = 'b'; |
||Query OK, 1 row affected (0.01 sec)|
|mysql> insert into tb values (4,'d', 't4');||
|Query OK, 1 row affected (0.01 sec)||

##### scene3

| session1 | session2 |
| :------ | :------ |
| mysql> begin; | mysql> begin; |
| |mysql> update tb set tag = 't' where id = 3; |
||Query OK, 1 row affected (0.01 sec)|
|mysql> insert into tb values (4,'d', 't4');||
|Query OK, 1 row affected (0.01 sec)||

以上，scene1复现了线上的情况，对比scene2/scene3又看到了不同的情况，很惊讶吧，具体原理及解释，需要先看看mysql的锁设计。

### mysql 锁设计

![lock critical](/assets/images/lock_critical.jpg)

锁是针对并发访问相同资源而设计的，如上图。innodb引擎为了控制事务间不同级别下的隔离性，提了供不同类型不同粒度的锁，同一个sql语句加锁情况会不一样。下面具体讨论下mysql存储引擎innodb的锁，建议先阅读[【MySQL 加锁处理分析】](http://hedengcheng.com/?p=771)这篇文章，理解什么是快照读与当前读，二阶段锁。

#### 锁类型

从控制并发读写相同资源的角度来看，innodb主要有两种类型的锁，即共享锁（读锁）和排他锁（写锁）。

* 共享锁，允许拿到共享锁的事务去读一行记录
* 排他锁，允许拿到排他锁的事务去更新或删除一行记录

兼容矩阵如下：

| | X | S |
| :------| :------ |:------|:------|
|X|Conflict|Conflict|	
|S|Conflict|Compatible|

#### 锁粒度

为提高事务并发读写数据的能力及实现数据库不同的隔离级别，innodb设计了不同粒度的锁，主要有两种：

* RECORD LOCK：行级锁，锁对象是innodb的索引记录，可以是一条记录，也可以是一个范围(gap lock, next-key lock都属于范围锁)。当待锁定记录所在表没有显式索引时，mysql把锁建立在隐藏的聚簇索引上。
* TABLE LOCK：表级锁，控制表级别的并发。需要注意的是 [Intention Locks 被归类为表级锁](https://dev.mysql.com/doc/refman/5.5/en/innodb-locking.html#innodb-intention-locks)，但我的理解是意向锁仍然用于控制行记录并发更新，由于表变更操作会影响行记录的变更，因此需要一种新的锁来衔接表锁和行锁，于是有了意向锁，表锁释放后，唤醒持有意向锁阻塞的线程（理解不一定对，请指正）。
 
需要特别注意的是，gap锁实现和传统意义上的锁有点不一样，gap锁的作用是防止其他线程往一个范围里插入满足条件的新纪录，gap锁之间不一定是冲突的。



#### 锁使用时机
innodb的这些锁什么时候被使用到？我们常见的CRUD操作，会加什么锁？如上面提到，不同隔离级别、where条件中使用不同索引，都会导致同一个sql请求获取不同的锁。

另外需特别注意，被加锁的记录与where条件是否匹配并没有关系：

> A locking read, an UPDATE, or a DELETE generally set record locks on every index record that is scanned in the processing of the SQL statement. It does not matter whether there are WHERE conditions in the statement that would exclude the row. 

以下列举常见的几种情况：

* INSERT：为防止其他线程插入相同记录，会在待插入记录主键上获取排它锁；如果是rr隔离级别，会提前获取待插入记录的 intent gap lock，避免幻读。
* SELECT：普通的 SELECT FROM 不会加任何锁，也会忽略任何已经加上的锁，除非是SERIALIZABLE隔离级别。SELECT FROM LOCK IN SHARE MODE 会获取共享 next-key lock，SELECT FROM FOR UPDATE 会获取排他的 next-key lock。
* UPDATE：设置意向排它锁，同时针对扫描的记录，设置排他的 next-key lock。
* DELETE：DELETE FROM 对扫描到的所有记录设置 next-key lock。

以上SELECT/UPDATE/DELETE，都要排除 where 条件里包含主键的情况，如果包含主键，则仅仅会锁住一条记录，而不是加 next-key lock。

#### 问题解释
innodb索引记录是按照B+树组织的，可参考此篇[文章](http://blog.lastww.com/2015/01/05/b-tree)，将上述记录按照B+数的组织方式拆分，假设拆分成如下几个范围：

    （-∞,1]，(1,2]，(2,3]，(3,+∞）

* scene1中session2事务update语句持有了(2,3], (3, 正无穷)的next-key lock，导致session1无法获取插入意向锁而阻塞
* scene2中session2事务，由于where条件包含了idx_name的非唯一性索引，update语句持有了(1, 2]，(2, 3]的next-key lock，而 session1插入的是大于3的记录，故可以插入
* scene3中session2事务update语句持有了primary key = 3的行锁，不影响session1获取意向锁

#### 为避免锁超时，使用事务时的注意点
* 事务里不要包含耗时的操作，大事务转成多个小事务；涉及较耗时的io操作移到事务外进行，减少锁定的资源和时间。
* 尽可能让事务里的检索、更新操作使用到索引，避免innodb无法使用到索引进而锁全表。
* 合理设计索引，尽可能是通过索引缩小innodb的锁定范围。
* 尽量避免范围查询、更新操作。

### 参考
1、[InnoDB Locking](https://dev.mysql.com/doc/refman/5.5/en/innodb-locking.html)

2、[Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/5.6/en/innodb-locks-set.html)

3、[MySQL 5.5 InnoDB 锁等待](https://dbarobin.com/2015/01/27/innodb-lock-wait-under-mysql-5.5/)

4、[MySQL 加锁处理分析](http://webcache.googleusercontent.com/search?q=cache:lHzy2YgeTOwJ:hedengcheng.com/%3Fp%3D771+&cd=4&hl=zh-CN&ct=clnk&gl=hk)

5、[MySQL数据库InnoDB存储引擎中的锁机制](http://www.cnblogs.com/kesongbing/archive/2012/09/24/2700742.html)

6、[Lock Basic Concept](https://pages.mtu.edu/~shene/NSF-3/e-Book/MUTEX/locks.html)
