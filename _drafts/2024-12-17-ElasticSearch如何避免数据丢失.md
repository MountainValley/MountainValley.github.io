

## 如何保证数据不丢失
1. index.translog.durability设置为request（某些版本下默认为request）
2. 副本数量大于1
3. 不同节点应在不同物理机上


## 背景知识：事务日志持久化和同步方式配置：
index.translog.durability

> By default, index.translog.durability is set to request meaning that Elasticsearch will only report success of an index, delete, update, or bulk request to the client after the translog has been successfully fsynced and committed on the primary and on every allocated replica. If index.translog.durability is set to async then Elasticsearch fsyncs and commits the translog only every index.translog.sync_interval which means that any operations that were performed just before a crash may be lost when the node recovers.

> index.translog.durability
> Whether or not to fsync and commit the translog after every index, delete, update, or > bulk request. This setting accepts the following parameters:
> 
> request
> (default) fsync and commit after every request. In the event of hardware failure, all > acknowledged writes will already have been committed to disk.
> async
> fsync and commit in the background every sync_interval. In the event of a failure, all > acknowledged writes since the last automatic commit will be discarded.
