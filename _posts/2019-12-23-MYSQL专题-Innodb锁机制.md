---
layout:     post
title:	MYSQL专题
subtitle: 	Innodb锁机制
date:       2019-12-23
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 数据库
---

# Innodb锁机制

数据库使用锁是为了支持更好的并发，提供数据的完整性和一致性。InnoDB是一个支持行锁的存储引擎，锁的类型有：共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）。为了提供更好的并发，InnoDB提供了非锁定读：不需要等待访问行上的锁释放，读取行的一个快照。该方法是通过InnoDB的一个特性：MVCC来实现的。

**InnoDB有三种行锁的算法：**

1，Record Lock：单个行记录上的锁。

2，Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况 。

3，Next-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

### 例子：

```mysql
create table t(a int,key idx_a(a))engine =innodb;
insert into t values(1),(3),(5),(8),(11);
select * from t;
-- Section A
start transaction;
select * from t where a = 8 for update;
-- Section B
begin;
select * from t;
insert into t values(2);
insert into t values(4);
insert into t values(6);
-- [SQL]
-- insert into t values(6);
-- [Err] 1205 - Lock wait timeout exceeded; try restarting transaction
```

- 为什么要针对查询操作锁定一个范围？而不是锁定一个行？
- 这个范围包含左右边界值吗？ 为什么？
  - 锁定是包含左边的值，不包含右边的值。因为隐藏的ROWID是递增的。例如上面的例子：对行 a=8 ，其实是a=8的前一个键值到后一个键值，[5,11)，不包含后一个键值的一个范围，5,6,7,9,10这些值的插入将被阻塞住。
- 幻读是怎么解决的？
  - 加锁： 间隙锁（不包含边界）和next-key locking锁（包含左边界不包含有边界）

**问题：**

**为什么section B上面的插入语句会出现锁等待的情况**？InnoDB是行锁，在section A里面锁住了a=8的行，其他应该不受影响。why？

**分析：**

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围（GAP）。上面索引值有1，3，5，8，11，其记录的GAP的区间如下：是一个**左开右闭**的空间（原因是默认主键的有序自增的特性，结合后面的例子说明）

（-∞,1]，(1,3]，(3,5]，(5,8]，(8,11]，(11,+∞）

特别需要注意的是，InnoDB存储引擎还会对辅助索引下一个键值加上gap lock。如上面分析，那就可以解释了。

```
root@localhost : test 10:56:29>select * from t where a = 8 for update;
+------+
| a    |
+------+
|    8 |
+------+
row in set (0.00 sec)
```

该SQL语句锁定的范围是（5,8]，下个下个键值范围是（8,11]，所以插入5~11之间的值的时候都会被锁定，要求等待。即：插入5，6，7，8，9，10 会被锁住。插入非这个范围内的值都正常。

因为例子里没有主键，所以要用隐藏的ROWID来代替，数据根据Rowid进行排序。而Rowid是有一定顺序的（自增），所以其中11可以被写入，5不能被写入，不清楚的可以再看一个有主键的例子：

### 有主键的例子

```mysql
create table t(id int,name varchar(10),key idx_id(id),primary key(name))engine =innodb;
insert into t values(1,'a'),(3,'c'),(5,'e'),(8,'g'),(11,'j');  
select @@global.tx_isolation, @@tx_isolation;   
select * from t;
-- sectiion A
start transaction;   
delete from t where id=8;

-- section B
select @@global.tx_isolation, @@tx_isolation;
start transaction; 
select * from t;
-- 
insert into t(id,name) values(6,'f');
insert into t(id,name) values(5,'e1');
insert into t(id,name) values(7,'h');
insert into t(id,name) values(8,'gg');
insert into t(id,name) values(9,'k');
insert into t(id,name) values(10,'p');
insert into t(id,name) values(11,'iz');
-- 以上都被阻塞住了
insert into t(id,name) values(5,'cz');
insert into t(id,name) values(11,'ja');
-- 说明了 gap lock 和 next-key locking锁?
```



**分析：**因为会话1已经对id=8的记录加了一个X锁，由于是RR隔离级别，INNODB要防止幻读需要加GAP锁：即id=5（8的左边），id=11（8的右边）之间需要加间隙锁（GAP）。这样[5,e]和[8,g]，[8,g]和[11,j]之间的数据都要被锁。上面测试已经验证了这一点，根据索引的有序性，数据按照主键（name）排序，后面写入的[5,cz]（[5,e]的左边）和[11,ja]（[11,j]的右边）不属于上面的范围从而可以写入。

另外一种情况，把name主键去掉会是怎么样的情况？有兴趣的同学可以测试一下。



### **继续：**插入超时失败后，会怎么样？

超时时间的参数：innodb_lock_wait_timeout ，默认是50秒。
超时是否回滚参数：innodb_rollback_on_timeout 默认是OFF。

```mysql
create table t(a int,key idx_a(a))engine =innodb;
insert into t values(1),(3),(5),(8),(11);
-- section A
start transaction;
select * from t where a = 8 for update;
-- section B
start transaction;
insert into t values(12);
insert into t values(10);
select * from t;
-- [SQL]
-- insert into t values(10);
-- [Err] 1205 - Lock wait timeout exceeded; try restarting transaction
```

经过测试，不会回滚超时引发的异常，当参数innodb_rollback_on_timeout 设置成ON时，则可以回滚，会把插进去的12回滚掉。

下图中，innodb_rollback_on_timeout 为ON，12被回滚掉了。

![1570521891245](/..\img\1570521891245.png)

默认情况下，InnoDB存储引擎不会回滚超时引发的异常，除死锁外。

既然InnoDB有三种算法，那Record Lock什么时候用？还是用上面的列子，把辅助索引改成唯一属性的索引。

### Record Lock实践

```mysql
create table t(a int primary key)engine =innodb;
insert into t values(1),(3),(5),(8),(11);
select * from t;
-- section A
start transaction;
select * from t where a = 8 for update;
-- section B
start transaction;
insert into t values(6);
insert into t values(7);
insert into t values(9);
insert into t values(10);
```

**问题：**

为什么section B上面的插入语句可以正常，和测试一不一样？

**分析：**

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围，按照这个方法是会和第一次测试结果一样。**但是，当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。**

注意：通过主键或则唯一索引来锁定不存在的值，也会产生GAP锁定。即

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `name` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
insert into t values(1,'a'),(3,'c'),(5,'e'),(8,'g'),(11,'j');   
-- section A
start transaction;
select * from t where id = 15 for update;
-- section B
insert into t(id,name) values(10,'k');
-- 以上可以执行
insert into t(id,name) values(12,'k');
insert into t(id,name) values(16,'kxx');
insert into t(id,name) values(160,'kxx');
-- 以上阻塞
```

如何让测试一不阻塞？可以显式的关闭Gap Lock：

1：把事务隔离级别改成：Read Committed，提交读、不可重复读。SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

2：修改参数：innodb_locks_unsafe_for_binlog 设置为1。

