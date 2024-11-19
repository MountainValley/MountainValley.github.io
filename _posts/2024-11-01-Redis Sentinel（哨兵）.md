---
title: Redis Sentinel（哨兵）
date: 2024-11-01 20:30:00 +0800
categories: [nosql, redis]
tags: [redis,nosql]     
---

# Redis Sentinel（哨兵）
Redis Sentinel 是 Redis 提供的高可用性解决方案，用于监控 Redis 主从集群，自动故障转移，并提供服务发现功能。

## 主要能力：
- 监控：Sentinel 会定期检查主节点和从节点的状态，以确保它们的可用性。如果 Sentinel 发现某个节点不可用，它会标记该节点为故障。
- 自动故障转移：当 Sentinel 发现主节点故障时，它会自动将一个从节点提升为新的主节点，确保服务的连续性。
- 通知：客客户端可以使用Sentinel作为与redis兼容的Pub/Sub服务器(但不能使用PUBLISH)，以便对通道进行SUBSCRIBE或PSUBSCRIBE，并获得有关特定事件的通知。channel的名字和事件名保持一致比如通过SUBSCRIBE +sdown 可以监听节点被标记为sdown（主观下线）事件
- 服务发现：Sentinel 提供 API，客户端可以通过这些 API 查询当前的主节点地址，以便自动适配主从结构的变化。

## 使用示例：
1. 配置Redis主从复制实现1主多从复制。
2. 启动奇数台（3台以上）Redis哨兵（实际上还是redis节点只不过启动命令不一样）
   1. 启动命令
   ```bash
   redis-sentinel /sentinel.conf
   ```
   或者也可以使用redis-server命令以哨兵模式启动
   ```bash
   redis-server /sentinel.conf --sentinel
   ```
   2. sentinel.conf配置文件介绍
   ```conf
    # 哨兵端口（用于客户端连接和多个sentinel之间交换故障数据、投票协商等重要操作）
    port 26379

    # 监视设置，所有哨兵皆一致，都指向 Master。
    sentinel monitor mymaster <master-ip> <master-port> <quorum>
    sentinel parallel-syncs mymaster 1
    sentinel down-after-milliseconds mymaster 30000
    sentinel failover-timeout mymaster 180000

    # 监视设置可以有多个以实现同一个sentinel集群监视不同的master
    sentinel monitor anotherMaster <master-ip> <master-port> <quorum>
    sentinel parallel-syncs anotherMaster 1
    sentinel down-after-milliseconds anotherMaster 30000
    sentinel failover-timeout anotherMaster 180000
   ```
   - mymaster：指定主节点的名称。
   - <master-ip> 和 <master-port>：主节点的 IP 地址和端口。
   - quorum：表明需要多少个 Sentinel 节点确认主节点故障才能进行故障转移。
   - down-after-milliseconds：主节点在多长时间内未响应后被视为故障。
   - failover-timeout：故障转移超时。
   - parallel-syncs：在故障转移后，有多少个从节点可以并行同步数据。

3. 可以使用jedis或redisson客户端的哨兵模式，客户端将从sentinel获取master的连接信息，读写命令都发送到master节点。当发生故障转移时客户端将收到通知并更新本地master连接信息。

## sentinel互相发现和状态更新
上面的示例中每个sentinel配置文件中只配置了master的连接信息，那么多个sentinel之间如何发现彼此又是如何同步master-slave节点信息的呢

每个sentinel启动时都会首先连接到配置的master节点上，然后利用pub/sub功能没2秒一次发布自己的相关信息（ip、port和runid等）到频道：__sentinel__:hello。同时监听该频道以获取其他sentinel发送的消息。这样后面有新的sentinel加入时之前已经在监听该频道的sentinel将会得到新节点的连接信息，并且最长2秒后新加入的sentinel也可以通过监听__sentinel__:hello频道消息获取其他sentinel信息。

上述消息不仅包含sentinel节点信息还包含master节点的完整配置信息（sentinel通过向主节点发送INFO命令获取），例如master状态、slave列表等信息，用来实现master数据同步。当sentinel获取到slave列表也会连接到所有slave上并按照上述方式发送和监听频道：__sentinel__:hello
