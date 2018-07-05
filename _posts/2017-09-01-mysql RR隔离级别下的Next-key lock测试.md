---
layout:       post
title:        "MySQL RR隔离级别下的Next-key lock测试"
subtitle:     "MySQL RR隔离级别下的Next-key lock测试"
date:         2017-09-01 12:00:00
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - MySQL
    - 锁
---

## 0x01 测试环境
__操作系统__：centos7<br>
__数据库版本__：mysql 5.7.19<br>
__存储引擎__：innodb<br>
__隔离级别__：Repeatable Read<br>
__表定义语句__：
```sql
CREATE TABLE `tuser` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(12) DEFAULT NULL,
  `test` bigint(20) NOT NULL,
  `remark` varchar(24) DEFAULT NULL,
  `createtime` datetime DEFAULT NULL,
  `updatetime` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_heihei` (`test`)
) ENGINE=InnoDB AUTO_INCREMENT=10001 DEFAULT CHARSET=utf8
```
__索引__：test列。

## 0x02 第一个测试步骤
* 向表中插入10000条数据。
* set autocommit=0;
* 开启两个事务
* 第一个事务执行

		update tuser set name='xxx' where test > 5000;
		
* 第二个事务执行

		insert into tuser (id, name, test) values (10001, 'xxx', 5001);
		
* 查询information_schema.INNODB_LOCKS可见

		*************************** 1. row ***************************
			lock_id: 96279:150:76:1
		lock_trx_id: 96279
		  lock_mode: X
		  lock_type: RECORD
		 lock_table: `test_batch`.`tuser`
		 lock_index: PRIMARY
		 lock_space: 150
		  lock_page: 76
		   lock_rec: 1
		  lock_data: supremum pseudo-record
		*************************** 2. row ***************************
			lock_id: 96278:150:76:1
		lock_trx_id: 96278
		  lock_mode: X
		  lock_type: RECORD
		 lock_table: `test_batch`.`tuser`
		 lock_index: PRIMARY
		 lock_space: 150
		  lock_page: 76
		   lock_rec: 1
		  lock_data: supremum pseudo-record
		2 rows in set, 1 warning (0.00 sec)
		
lock_data的值为supremum pseudo-record，这代表比索引中所有值都大，但却不存在索引中，相当于最后一行之后的gap lock。

## 0x03 第二个测试步骤
* 向表中插入10000条数据。
* set autocommit=0;
* 开启两个事务
* 第一个事务执行

		update tuser set name='xxx' where test = 5000;
		
* 第二个事务执行

		insert into tuser (id, name, test) values (10001, 'xxx', 5000);
		
* 查询information_schema.INNODB_LOCKS可见

		*************************** 1. row ***************************
			lock_id: 96283:150:52:626
		lock_trx_id: 96283
		  lock_mode: X,GAP
		  lock_type: RECORD
		 lock_table: `test_batch`.`tuser`
		 lock_index: idx_test
		 lock_space: 150
		  lock_page: 52
		   lock_rec: 626
		  lock_data: 5001, 5002
		*************************** 2. row ***************************
			lock_id: 96281:150:52:626
		lock_trx_id: 96281
		  lock_mode: X,GAP
		  lock_type: RECORD
		 lock_table: `test_batch`.`tuser`
		 lock_index: idx_test
		 lock_space: 150
		  lock_page: 52
		   lock_rec: 626
		  lock_data: 5001, 5002
		2 rows in set, 1 warning (0.00 sec)
		
可以看到lock_mode为__X,Gap__，这代表了update语句拿到了5000的gap锁。

## 0x03 第三个测试步骤
* 向表中插入10000条数据。
* set autocommit=0;
* 开启两个事务
* 第一个事务执行

		update tuser set name='xxx' where test > 5001;
		
* 第二个事务执行

		insert into tuser (id, name, test) values (NULL, 'xxx', 5001);

* 查询information_schema.INNODB_LOCKS可见

		*************************** 1. row ***************************
		    lock_id: 249524:278:4:3
		lock_trx_id: 249524
		  lock_mode: X,GAP
		  lock_type: RECORD
		 lock_table: `test`.`tuser`
		 lock_index: idx_heihei
		 lock_space: 278
		  lock_page: 4
		   lock_rec: 3
		  lock_data: 5002, 2
		*************************** 2. row ***************************
		    lock_id: 249520:278:4:3
		lock_trx_id: 249520
		  lock_mode: X
		  lock_type: RECORD
		 lock_table: `test`.`tuser`
		 lock_index: idx_heihei
		 lock_space: 278
		  lock_page: 4
		   lock_rec: 3
		  lock_data: 5002, 2

可以看到事务一拿到了Next-Key Lock锁，而事务二只拿了X锁。