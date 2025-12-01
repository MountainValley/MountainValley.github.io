---
title: 分布式事务框架 Apache Seata
date: 2024-12-10 16:30:00 +0800
categories: [微服务开发]
tags: [分布式事务,spring-cloud-alibaba] 
---

## 分布式事务相关概念

| 概念                                      | 职责                                                                                 |
| ----------------------------------------- | ------------------------------------------------------------------------------------ |
| TM（Transaction Manager，事务管理器）     | 负责全局事务的管理，包括全局事务的创建、提交和回滚。                                 |
| TC（Transaction Coordinator，事务协调器） | 负责保存和维护全局事务的状态，并协调各个分支事务的执行。                             |
| RM（Resource Manager，资源管理器）        | 负责具体资源（如数据库、消息队列等）的分支事务管理，包括分支事务的注册、提交和回滚。 |


### Seata分布式事务架构
![Seata分布式事务架构](/assets/images/seata/image.png)

### Seata分布式事务底层依赖的相关库表

| 表名             | 部署位置       | 模式 | 说明                                             |
| ---------------- | -------------- | ---- | ------------------------------------------------ |
| UNDO_LOG         | 每个业务库一份 | AT   | RM负责在执行实际的写（insert\delete\update）语句 |
| global_table     |                |      | 全局事务信息表                                   |
| branch_table     |                |      | 分支事务表                                       |
| lock_table       |                |      | 分布式事务锁表                                   |
| distributed_lock |                |      |                                                  |


```SQL
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL COMMENT '全局事务 ID',
    `transaction_id`            BIGINT COMMENT '事务 ID',
    `status`                    TINYINT      NOT NULL COMMENT '事务状态 (0: 开启, 1: 提交, 2: 回滚等)',
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL COMMENT '分支事务 ID',
    `xid`               VARCHAR(128) NOT NULL COMMENT '全局事务 ID',
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32) COMMENT '资源组 ID',
    `resource_id`       VARCHAR(256) COMMENT '资源 ID',
    `branch_type`       VARCHAR(8) COMMENT '分支事务类型 (AT, TCC 等)',
    `status`            TINYINT COMMENT '分支事务状态 (0: 开始, 1: 提交, 2: 回滚等)',
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL COMMENT '锁的唯一标识（资源 + 主键）',
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL COMMENT '分支事务 ID',
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```