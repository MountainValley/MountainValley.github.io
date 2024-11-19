---
title: MySQL Multi-Version Concurrency Control (MVCC)
date: 2021-12-10 18:49:20 +0800
categories: [数据库, MySQL]
tags: [mysql]     
---

# MySQL Multi-Verison Concurrency Control (MVCC，多版本并发控制)



### 什么是MVCC

MVCC是数据库管理系统常用的一种并发控制（MySQL、Oracle、SQL Server等主流数据库都在使用MVCC），**通过保存数据在某个时间点的快照**让select语句直接读取指定版本的值已避免加锁，从而提高并发读写性能。

### MVCC的两种读

在MVCC并发控制中，读可以分为两类，**快照读**和**当前读**。

- 快照读
  - 读取的是记录数据的**可见版本**（可能是过期的数据），不用加锁
  - 普通的select使用该读取方式
- 当前读
  - 读取的是记录数据的**最新版本**，并且当前读**返回的记录都会加上锁**，保证其他事务不会再并发的修改这条记录
  - select ... lock in share mode
  - select ... for update
  - insert
  - update
  - delete
  - 以上查询使用当前读


### MySQL在什么情况下会使用MVCC

InnoDB在RC或者RR隔离级别下通过快照读读取数据时用到

### MVCC实现原理

#### InnoDB数据表隐藏列

InnoDB数据行包含了一些额外的隐藏字段（对查询不可见），其中DATA_TRX_ID、DATA_ROLL_PTR、DELEETE BIT用于MVCC的实现。

| 字段        | 长度          | 作用                                                                                                                                                              |
| :---------- | :------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DB_TRX_ID   | 6字节（Byte） | InnoDB维护了全局的事务ID，每次开启一个新事务时该全局事务ID会加一。数据行上记录的DATA_TRX_ID为该行数据最新一次被添加或修改（删除也视为一种修改）时的所在事务的ID。 |
| DB_ROLL_PTR | 7字节（Byte） | 指向rollback segment里的一条undo log记录。一旦数据被修改该undo log记录存储的内容可用于重新构建该行记录到被修改之前的版本                                          |
| DELETE BIT  | 1 bit         | 删除标记                                                                                                                                                          |

#### 全局事务链表（当前活跃链表，trx_sys）

MySQL中的事务ID从开始到提交之前这段时间会被保存到一个全局事务链表trx_sys中。这是一个基本的链表结构。一旦事务提交，该事务ID将从trx_sys中移除。

#### Read View 结构
Read View数据结构中几个比较重要的成员
| 成员           | 描述                                                                                                |
| -------------- | --------------------------------------------------------------------------------------------------- |
| low_limit_id   | 低限制事务 ID：所有事务 ID 小于 low_limit_id 的事务都已提交，其产生的修改对当前事务可见。           |
| trx_ids        | Reas View创建时当前系统中正在运行的事务 ID 列表（活跃事务的事务 ID 集合）。                         |
| up_limit_id    | 高限制事务 ID：表示当前活跃事务中最小的事务 ID，所有事务 ID 大于等于 up_limit_id 的事务修改不可见。 |
| creator_trx_id | Read View创建时所在的当前事务ID                                                                     |

#### undo log

**undo log** 中存储的是历史版本的数据。当一个事务进行查询时如果数据库最新的记录对当前查询不可见，那么会通过当前记录的隐藏列DB_ROLL_PTR的值去rollback segment里取出对应的一条undo log以获取当前记录的上一版本数据，然后判断其上一版本数据是否可见，如果不可见则继续根据上一版本数据的DB_ROLL_PTR值在undo log链中继续寻找直至找到最大的某个可见版本进行返回，如果找不到则返回空。

rollback segment中的undo logs分为两类，分别是insert undo logs和update undo logs。insert undo logs 的唯一作用就是用来回滚事务的时候删除记录行，一旦事务提交就可以被删除了。而update undo logs只有当所有活跃事务中都不再需要使用该update log来获取其对应的历史快照数据时才能被删除。

#### 数据行修改过程

1. 对该行数据加排它锁（X）
2. 如果加锁成功则将该行数据拷贝的undo log中记录历史版本快照，否则等待加锁
3. 拷贝到undo log中后修改数据行信息，并更新数据行隐藏字段DB_TRX_ID为当前事务ID，DB_ROLL_PTR为指向上一步undo log快照记录的值。（然后还会将修改后的数据行写入redo log中）
4. 提交事务并释放排它锁

#### MVCC快照读具体流程

<img src="/assets/images/mvcc/MySQL MVCC 快照读详细流程图.png" alt="MVCC 快照读具体流程" style="zoom:87%;" />

#### MVCC可见性判断算法

对数据行进行可见性判断主要基于数据行的DB_TRX_ID以及在SELECT语句执行前生成的read view。（RC级别是是事务内每一次SELECT语句执行前都会重新生成新的read view，而RR级别则只在第一次执行SELECT语句的时候生成一次）

当事务尝试读取某条记录时，InnoDB 通过 ReadView 判断该记录的版本是否可见：

- 事务版本创建 ID 小于 low_limit_id：该事务早已提交，对当前事务可见。
- 事务版本创建 ID ≥ up_limit_id： 该事务尚未提交，对当前事务不可见。
- 事务版本创建 ID 位于 trx_ids 中：该事务正在执行，对当前事务不可见。
- 事务版本创建 ID 是 creator_trx_id：当前事务自身的更改版本始终可见。


**<font style="color:orange;">Q:RR级别下MVCC是否能避免幻读</font>**

一般正常业务场景下不会发生幻读。但是加入同一个事务先进行一次快照读，后面相同的查询条件下进行一次当前读。如果两个读操作直接有其他事务插入了数据会出现幻读。为了避免此情况一个事务内相同条件多次查询要么都是当前读要么都是快照读。如何混搭的话请把当前读放在前面，因为当前读是通过加临键锁的方式避免的幻读。












