# ZooKeeper

分布式应用协调服务


## 目录

- ZooKeeper 使命、基础概念

- ZooKeeper 内部运行原理

- ZooKeeper 的源码实现

- ZooKeeper 实践采坑和优化



## 使命、基础概念

### 使命

#### 专注分布式协作

#### 开发人员自己决定具体的协同任务

#### 分布式锁，分布式队列，主-从模式下的选举

#### 不适合海量数据存储


### 基础概念

#### 数据结构 (znode)

![image](https://zookeeper.apache.org/doc/current/images/zknamespace.jpg)

持久节点、临时节点、有序节点

#### api

```
create /path data 创建一个名为/path的znode节点，并包含数据data。

delete /path 删除名为/path的znode。

set /path data

get /path

ls /path
```

#### 组成模块

![image](https://zookeeper.apache.org/doc/current/images/zkservice.jpg)

![image](https://zookeeper.apache.org/doc/current/images/zkcomponents.jpg)


#### 运行模式

```

standalone 和 quorum

客户端与服务端的会话(session)使用 TCP 协议

同一 session 中的请求以 FIFO 顺序执行

```


## 内部运行原理

- zxid
- Leader 选举
- Zab 协议
- 请求处理链
- 本地存储

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

![集群模式 Leader 请求处理链](https://user-images.githubusercontent.com/8939151/83320905-2e9f6b00-a27e-11ea-9b3f-ee78e4eb3912.png)

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
Follower 服务器是先从 FollowerRequestProcessors 处理器开始，该处理器接收并处理客户端请求，FollowerRequestProcessors 处理器之后转发请求给 CommitRequestProcessor，同时也会转发写请求到群首服务器。CommitRequestProcessor 会直接转发读取请求到 FinalRequestProcessor 处理器，而且对于写请求，在转发前会等待提交事务。而群首接收到一个新的写请求时会生成一个提议，之后转发到追随者服务器，在收到一个提议，追随服务器会发送这个提议到 SyncRequestProcessor，SendRequestProcessor 会向群首发送确认消息。

当群首服务器接收到足够多确认消息来提交这个提议是，群首就会发送提交事务消息给追随者，当收到提交的事务消息时，追随者就通过 CommitRequestProcessor 处理器进行处理。为了保证执行的顺序，CommitRequestProcessor 处理器会在收到一个写请求处理器时暂停后续的请求处理。

对于观察者服务器不需要确认提议消息，因此观察者服务器并不需要发送确认消息给群首服务器，一般情况下，也不用持久化事务到磁盘。对于观察者服务器是否持久化事务到磁盘，以便加速观察者服务器的恢复速度，可以根据具体情况决定。

```

### 本地存储

SyncRequestProcessor 处理器是用于处理提议写入的日志和快照。

#### 事务日志

```
日志和磁盘的使用

服务器通过事务日志来持久化事务。在接受一个提议时，一个服务器就会将提议的事务持久化到事务日志中，该事务日志保存在服务器本地磁盘中，而事务将会按照顺序追加其后。写事务日志是写请求操作的关键路径，因此 ZooKeeper 必须有效处理写日志问题。在持久化事务到磁盘时，还有一个重要说明：现代操作系统通常会缓存脏页（Dirty Page），并将他们异步写入磁盘介质。然而，我们需要在继续之前，要确保事务已经被持久化。因此我们需要冲刷（Flush）事务到磁盘介质。

冲刷在这里就是指我们告诉操作系已经把脏页写入到磁盘，并在操作完成后返回。同时为了提高 ZooKeeper 系统的运行速度，也会使用组提交和补白的。其中组提交是指一次磁盘写入时追加多个事务，可以减少磁盘寻址的开销。补白是指在文件中预分配磁盘存储块。

```

#### 快照

```
快照是 ZooKeeper 数据树的拷贝副本，每一个服务器会经常以序列化整个数据树的方式来提取快照，并将这个提取的快照保存到文件。服务器在进行快照时不需要进行协作，也不需要暂停处理请求。因此服务器在进行快照时还会继续处理请求，所以当快照完成时，数据树可能又发生了变化，称为快照是模糊的，因为它们不能反映出在任意给定的时间点数据树的准确的状态。
```

## 源码实现

主要看数据结构，类结构，线程模型，流程等方面

### 服务端

ZooKeeper 服务启动方式分为三种，单机模式、伪分布式模式、分布式模式。这里主要研究分布式模式，

主要经过 Leader 选举，集群数据同步，启动服务器。

```
分布式模式下的启动过程包括如下阶段，

- 解析 config 文件；

- 数据恢复；

- 监听 client 连接(但还不能处理请求)；

- bind 选举端口监听 server 连接；

- 选举；

- 初始化 ZooKeeperServer；

- 数据同步；

- 同步结束，启动 client 请求处理能力。
```

### 客户端

```
从整体看，客户端启动的入口时 ZooKeeperMain，在 ZooKeeperMain 的 run()中，创建出控制台输入对象（jline.console.ConsoleReader），然后它进入 while 循环，等待用户的输入。同时也调用 connectToZK 连接服务器并建立会话（session），在 connect 时创建 ZooKeeper 对象，在 ZooKeeper 的构造函数中会创建客户端使用的 NIO socket，并启动两个工作线程 sendThread 和 eventThread，两个线程被初始化为守护线程。

sendThread 的 run()是一个无限循环,除非运到了 close 的条件,否则他就会一直循环下去,比如向服务端发送心跳,或者向服务端发送我们在控制台输入的数据以及接受服务端发送过来的响应。

eventThread 线程负责队列事件和处理 watch。

客户端也会创建一个 clientCnxn，由 ClientCnxnSocketNIO.java 负责 IO 数据通信。

客户端的场景说明（事务、非事务请求类型）。
```

### 服务端和客户端结合部分

```

会话（Session)

Client 建立会话的流程如下，

- 服务端启动，客户端启动；

- 客户端发起 socket 连接；

- 服务端 accept socket 连接，socket 连接建立；

- 客户端发送 ConnectRequest 给 server；

- server 收到后初始化 ServerCnxn，代表一个和客户端的连接，即 session，server 发送 ConnectResponse 给 client；

- client 处理 ConnectResponse，session 建立完成。

监视（Watch）

本小节主要看看 ZooKeeper 怎么设置监视和监控点的通知。ZooKeeper 可以定义不同类型的通知，如监控 znode 的数据变化，监控 znode 子节点的变化，监控 znode 的创建或者删除。ZooKeeper 的服务端实现了监视点管理器（watch manager）。

一个 WatchManager 类的实例负责管理当前已经注册的监视点列表，并负责触发他们，监视点只会存在内存且为本地服务端的概念，所有类型的服务器都是使用同样的方式处理监控点。

DataTree 类中持有一个监视点管理器来负责子节点监控和数据的监控。

在服务端触发一个监视点，最终会传播到客户端，负责处理传播的为服务端的 cnxn 对象（ServerCnxn 类），此对象表示客户端和服务端的连接并实现了 Watcher 接口。Watch.process 方法序列化了监视点事件为一定的格式，以便于网络传送。ZooKeeper 客户端接收序列化的监视点事件，并将其反序列化为监控点事件的对象，并传递给应用程序。

客户端 watcher 实现

在客户端 GetData 时，如果注册 watch 监控点到服务端，在 watch 的 path 的 value 变化时，服务端会通知客户端该变化。

在客户端的 GetData 方法中（ZooKeeper 类的 GetData）:

创建 WatchRegistration wcb= new DataWatchRegistration(watcher, clientPath)，path 和 watch 封装进了一个对象；

创建一个 request，设置 type 为 GetData 对应的数值；

request.setWatch(watcher != null)，setWatch 参数为一个 bool 值。

调用 ClientCnxn.submitRequest(...) , 将请求包装为 Packet，queuePacket()方法的参数中存在创建的 path+watcher 的封装类 WatchRegistration，请求会被 sendThread 消费发送到服务端。

```



## 实践采坑和优化



