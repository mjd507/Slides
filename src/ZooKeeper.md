# ZooKeeper

## 目录

- ZooKeeper 是什么

- ZooKeeper 的使用场景

- ZooKeeper 的内部原理

- ZooKeeper 是什么


## ZooKeeper 是什么

### 中间件服务，专注分布式应用协作。

### 基础概念

#### 数据结构 (znode)

![image](https://zookeeper.apache.org/doc/current/images/zknamespace.jpg)

```text
        类型：持久节点、持久有序节点、临时节点、临时有序节点
```

#### api

```
ls /path  查看当前路径下有哪些节点

create /path data 

get /path 

delete /path 

set /path data

```

#### 组成模块

![image](https://zookeeper.apache.org/doc/current/images/zkservice.jpg)

![image](https://zookeeper.apache.org/doc/current/images/zkcomponents.jpg)


## ZooKeeper 使用场景

### 配置中心

![config](https://picturebook-s3.alo7.com/static/share/zk-config.jpg)

### 分布式锁

![distrubute-lock](https://picturebook-s3.alo7.com/static/share/zk-lock.jpg)

### 分布式队列

![distrubute-queue](https://picturebook-s3.alo7.com/static/share/zk-queue.jpg)

### master-worker 协同

![master-worker](https://picturebook-s3.alo7.com/static/share/zk-master-worker.jpg)

### 集群管理

![cluster management](https://picturebook-s3.alo7.com/static/share/zk-cluster-manage.jpg)


## 内部运行原理

- zxid
- Leader 选举
- Zab 协议
- 请求处理链
- 数据存储

### zxid

```
zookeeper transaction id

long 型，32 位 epoch + 32 位 counter.

```

### Leader 选举

```
基于 TCP 投票， Vote(sid, zxid)

投票规则:

1. 比较 zxid(epoch + counter), 投值教大者.

2. 如果 zxid 相等，投 sid 较大者.

```

### Zab 协议

```
ZooKeeper Atomic Broadeast protocol 原子广播协议

zab 协议提交一个事务类似于两阶段提交

1. Leader 向所有 Follower 发送一个 PROPOSAL 消息 p

2. Follower 收到消息 p 后，ACK 通知 Leader 已接受该提案(proposal)

3. 当 Leader 收到过半(包括自己) 的 ack 消息后，发送消息通知 Follower 进行 COMMIT 

```

### 请求处理链


#### standalone 模式请求处理

```
PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor

1. PrepRequestProcessor 预处理(写请求生成zxid)
2. SyncRequestProcessor 数据落盘(日志+快照)
3. FinalRequestProcessor 返回数据 (写请求还会回写到内存)

```

#### quorum 模式 Leader 请求处理

![集群模式 Leader 请求处理链](https://picturebook-s3.alo7.com/static/share/zk-processer.png)

```

1. PrepRequestProcessor 为写请求生成 zxid
2. ProposalRequestProcessor 生成提议
   读写请求均会转发到 CommitRequestProcessor
   写请求还会转发给 SyncRequestProcessor
3. SyncRequestProcessor 持久化事务到磁盘
   执行完触发 AckRequestProcessor，给 Leader 响应.
4. CommitRequestProcessor 会将收到足够多的确认消息的提议进行提交。

```

#### quorum 模式 Follower 请求处理

```

FollowerRequestProcessor -> CommitProcessor 
读: -> FinalRequestProcessor
写: -> 转发给 Leader 

响应 Leader Proposal
SyncRequestProcessor -> SendAckRequestProcessor

```

### 数据存储


#### 内存数据

```
持久节点: 
ConcurrentHashMap<String, DataNode>();

临时节点:
ConcurrentHashMap<Long, HashSet<String>>();

```

#### 事务日志

```

事务日志文件固定为为 64MB，使用 0 补充空余部分

当不足 4kb 时，会预分配新的 64 MB 事务日志文件

使用组提交，高并发写入情况下，每 1000 次提交刷一次磁盘

```

#### 快照

```
基于当前的 DataTree 内存数据，不一定是最新数据

快照生成时机: countLog > snapCount/2 + randRoll(snapCount/2)

使用「异步线程」生成快照

```

## ZooKeeper 是什么

### 文件读写系统 + 事件通知机制



