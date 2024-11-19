---
title: Redis Cluster（集群）
date: 2024-11-01 20:30:00 +0800
categories: [nosql, redis]
tags: [redis,nosql]     
---

# Redis Cluster（集群）

## 什么是Redis Cluster？
当redis处理数据量规模较大时可以使用Redis Cluster对数据进行水平拆分。Redis Cluster将数据分片到2^14=16384个哈希槽(hash slot)上，Redis Cluster中每个Master只负责部分哈希槽，同时每个Master可配置一定数量的Slave以实现集群的高可用。

## Redis 如何启用Cluster模式？

### 前置条件：以cluster模式启动多个redis-server
在redis.conf中使用如下配置启用cluster
```conf
cluster-enabled yes
```
**注意：集群模式下最少应该要有3个master节点并最好每个master还应有至少一个slave**

### 方式一：初始化使用自动分配哈希槽机制启动Cluster模式
```bash
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1
```
选项：--cluster-replicas 1代表每个master需要有1个slave。
执行上述命令时redis-cli会响应一个推荐配置，我们输入yes后redis-cli将使用此配置类设置集群。当设置完毕后会响应：
```bash
[OK] All 16384 slots covered
```
此时6个节点被配置为3主3从的redis集群，且3个主节点完全覆盖了所有哈希槽。集群进入可用状态。

### 方式二：手动添加节点（必须启用了cluster模式的节点）并手动分配哈希槽
例如在上面集群基础上再添加一个master和一个slave节点。
```bash
# 添加master节点到集群
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000
```
上面add-node第一个地址是新节点地址，第二个地址是现有集群中任意一个节点地址。
```bash
# 添加slave节点到集群并指定其复制的master节点nodeid
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7000 --cluster-slave --cluster-master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e
```
节点成功加入集群后开始分配哈希槽。
```bash
redis-cli --cluster reshard 127.0.0.1:7000
```
通过连接集群内任意节点执行reshard操作即可，该命令会通过多次提问的方式引导用户输入配置数据以完成重新分配哈希槽的任务。例如，首先会询问需要移动的哈希槽数量、移动的目的地nodeid、从哪些源节点上移动。

## slot迁移过程对读写的影响
slot迁移过程中，源节点和目标节点均会记录迁移状态。当迁移过程中客户端向源节点请求数据是源节点检查数据是否仍在当前节点如果在则正常响应数据，如果已经移动则返回ASK重定向。这表示该键暂时移动到了目标节点，客户端需要向目标节点发送请求。这个过程是临时的，意味着在迁移完成之前，客户端需要继续使用 ASK 重定向来访问该键。一旦迁移完成，源节点会返回 MOVED 重定向，指示该槽及其数据已经永久移动到目标节点。此时客户端需要更新其内部的哈希槽和节点映射关系，以反映这一变化。因此使用开发良好的redis客户端情况下slot迁移过程对功能性是没有影响的。

## Redis Cluster使用注意事项
1. cluster模式下redis只使用第0号database
2. cluster模式下使用pub/sub，消息会在所有节点之间广播以避免消息无法送达的情况。