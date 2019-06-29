# Java 中间件

## 目录

- 服务中间件

- 数据库中间件

- 消息中间件


## 服务中间件

- 寻址路由

- 通信方式

- 序列化


### 寻址路由

#### 透明代理

- 硬件负载均衡设备

- LVS 等软件负载均衡

#### 服务注册中心


#### 常见的负载均衡算法

##### [轮询算法](https://github.com/weibocom/motan/blob/master/motan-core/src/main/java/com/weibo/api/motan/cluster/loadbalance/RoundRobinLoadBalance.java)

```html
按照固定的顺序，把可用的服务节点，挨个访问一次。

比如服务有 10 个节点，放到一个大小为 10 的数组，从序号为 0 的节点开始访问，
访问后序号自动加 1，以此类推。

能够保证所有节点被访问到的概率是相同的。

```

##### [加权轮询算法](https://github.com/weibocom/motan/blob/master/motan-core/src/main/java/com/weibo/api/motan/cluster/loadbalance/ConfigurableWeightLoadBalance.java)

```html
在轮询算法基础上，给每个节点一个权重，使每个节点被访问到的概率不同。

加权轮询算法是生成一个节点序列，该序列里有 n 个节点，n 
是所有节点的权重之和。每个节点出现的次数，就是它的权重值。

比如有三个节点：a、b、c，权重分别是 3、2、1，那么生成的序列就是 
{a、a、b、c、b、a}，这样的话按照这个序列访问，前 6 次请求就会
分别访问节点 a 三次，节点 b 两次，节点 c 一次。
从第 7 个请求开始，又重新按照这个序列的顺序来访问节点。


```

##### [一致性 hash 算法](https://github.com/weibocom/motan/blob/master/motan-core/src/main/java/com/weibo/api/motan/cluster/loadbalance/ConsistentHashLoadBalance.java)

```html
通过某个 hash 函数，把同一个来源的请求都映射到同一个节点上。

```

### 通信方式

#### bio

```html
基于流模型实现的，提供「同步、阻塞」方式。

在读取输入流或者输出流时，读写动作完成之前，线程会一直阻塞在那里。
```

##### 操作系统层图例

![image](http://loveshisong.cn/static/images/BIO.png)

##### 应用层图例

![image](http://icdn.apigo.cn/blog/javacore-io-005.png)


#### nio

```html
基于 Channel、Selector、Buffer 等实现多路复用，
提供「同步，非阻塞」IO 操作。

所有 socket 建立连接后，会注册到 Selector (多路复用器) 上。
```

##### 操作系统层图例一

![image](http://loveshisong.cn/static/images/NIO.png)

##### 操作系统层图例二

![image](http://loveshisong.cn/static/images/Multiplexing_IO.png)

##### 应用层图例

![image](http://icdn.apigo.cn/blog/javacore-io-006.png)


#### aio

```html
Java 1.7 之后引入，NIO 的升级版本，异步非堵塞的 IO 操作。

基于事件和回调机制实现的，操作之后会直接返回，不会堵塞，
当后台处理完成，会通知相应的线程进行后续的操作。
```

##### 操作系统层图例二

![image](http://loveshisong.cn/static/images/AIO.png)


### 序列化

#### json 
```json
{
  "strKey" : "outer-key",
  "intKey" : 1,
  "boolKey" : true,
  "innerObj" : {
    "key" : "inner-key",
    "val" : "inner-val"
  },
  "objList" : [ {
    "key" : "inner-key",
    "val" : "inner-val"
  }, {
    "key" : "inner-key",
    "val" : "inner-val"
  } ]
}
```

#### Java 序列化

```html
��sr/com.alo7.picturebook.toc.test.serial.TestEntityzH��3�
�LboolKeytLjava/lang/Boolean;LinnerObjt:Lcom/alo7/picturebook
/toc/test/serial/TestEntity$InnerObj;LintKeytLjava/lang/Integer;
LobjListtLjava/util/List;LstrKeytLjava/lang/String;xpsrjava.lang.
Boolean� r�՜��Zvaluexpsr8com.alo7.picturebook.toc.test.serial.
TestEntity$InnerObj���J��Lkeyq~LvaltLjava/lang/Object;xpt	
inner-keysrjava.lang.Integer⠤���8Ivaluexrjava.lang.Number
������xp{sq~srjava.util.ArrayListx����a�Isizexpwq~q~xtstr-key
```

#### xml 

```xml
<TestEntity>
  <strKey>outer-key</strKey>
  <intKey>1</intKey>
  <boolKey>true</boolKey>
  <innerObj>
    <key>inner-key</key>
    <val>inner-val</val>
  </innerObj>
  <objList>
    <objList>
      <key>inner-key</key>
      <val>inner-val</val>
    </objList>
    <objList>
      <key>inner-key</key>
      <val>inner-val</val>
    </objList>
  </objList>
</TestEntity>

```

#### Protocol Buffers

```html

  outer-key"
  inner-key  inner-val2
  inner-key  inner-val2
  inner-key  inner-val
```

#### 性能

![image](https://user-images.githubusercontent.com/8939151/57075093-61dc9700-6d18-11e9-81bc-6047a41849c9.png)


## 数据库中间件

- 读写分离

- 分库分表

- 设计思路

### 读写分离

适用于读多写少，减少主库压力

#### 问题

- 主从数据复制问题

- 应用对于数据源选择问题

#### 数据复制对策

- 放弃一致性要求，保证最终一致即可

- 通过程序，将读写操作均指向主数据库

#### 数据源选择对策

- 通过 proxy，由 proxy 来判断主库还是从库执行

- 应用内配置两种数据源，根据 sql 的类型（读/写）来调用对应的数据库


### 分库分表

可以只分库，或者只分表

#### 垂直拆分

在应用服务化之后，可以将相对内聚的微服务单独分配一个数据库，减轻单体数据库的压力

#### 水平拆分

当单表达到一定数据量时，可以通过增加一张相同字段的表，来分摊单表压力

#### 垂直拆分 问题

- 单机的 acid 无法保证

- join 操作无法执行

#### 垂直拆分 对策

- 分布式事务来保证，cap 理论，base 理论

- join 的问题
  - 可将原有的 sql，分割为两条
  - 在分库建表时，冗余一些必要字段。

#### 水平拆分 新增问题

- 主键问题

- SQL 路由问题

- 分页查询问题


#### 水平拆分 对策

- 主键问题：保证唯一性和连续性
  - 推荐 Twitter 的 Snowflake 算法
  - 生成的是 64 位唯一 Id（41 位的 timestamp + 10 位自定义的机器码 + 13 位累加计数器）

- SQL 路由问题：数据分片的时候按照指定的 key，路由存储到不到的表中。

#### 水平拆分 对策

- 分页查询问题：
  - 不需要排序，按等步长或等比例从多张表中获取
  - 需要排序，每张表都需获取足够查询页数的数据，在应用内合并排序，当页数过大时，负担越重，尽量避免。


## 消息中间件

- 发送和接受消息

- 分摊单个应用同时调用多个服务的压力

### 如何确保消息发送成功

#### 方法一

```java
void foo1() {
  // 业务操作：写数据库，调用服务等
  // 发送消息给中间件
}
```

#### 方法二

```java
void foo2() {
  // 发送消息给中间件
  // 业务操作：写数据库，调用服务等
}
```

#### 方法三

```java
void foo3() {
  // 发送消息给中间件，带个标记为待处理，并返回消息存储结果（成功/失败）
  // if(消息中间件返回 false) { 放弃业务处理，结束 }
  // 业务操作：写数据库，调用服务等，并返回业务接口（成功/失败）
  // 发送业务结果给消息中间件
  // （消息中间件收到业务结果：if(业务成功){ 更新消息状态为可发送，并进行调度 } else { 删除消息，结束 }）
}
```


### 如何解决与使用者的强依赖关系

#### 将本地磁盘作为一个消息存储的容器

![image](https://user-images.githubusercontent.com/8939151/59153124-19399b80-8a85-11e9-8aa4-7ce730dcdc80.png)

#### 图解

- 在应用内，通过消息中间件客户端，将应用内的消息写入本地磁盘

- 因为在应用内，可以做成一个本地事务，保证应用内的业务和消息发送一定是成功的

- 本地磁盘消息写入成功后，消息中间件客户端就可以通过（轮询本地消息等方式）控制将消息向服务端发送。


### 消息存储的几种方式

#### 3 种方式
- 实现基于文件的消息存储

- 采用数据库进行消息存储

- 基于双击内存的消息存储


### 如何确保消息投递(消费)成功

#### 消费成功的条件

- 当且仅当消费者显式的告诉中间件消费消费成功时，才算消费成功

#### 消费失败的闭环处理

- 将失败消息扔到一个队列中，记录扔进去的时间

- 用 job 每 10 分钟取一次失败消息，重新消费

- job 消费成功，从队列移除，消费失败，留在队列

- 24 小时，还没消费成功，发送报警邮件


### 重复消息的产生和方案

#### 应用端消息的重复发送

- 中间件消息存储失败，应用重复发送，正常行为

- 中间件存储成功，但因网络出现故障，或者负载高导致响应超时，导致应用消息重复发送

#### 解决办法

- 重试发消息时，使用同一个消息 id，而不用在消息中间件端产生消息 id


#### 中间件向外投递，产生重复

- 消息消费者出问题了，网络出问题了，或者处理时间较长，导致消息中间件重复投递

- 中间件收到结果了，但是自身挂了，或者消息状态更新时出故障了，导致重复投递

- 消息到达了消息存储，由中间件向外投递时，产生重复


#### 解决办法

- 让消息消费者来处理，要求消费者对消息的处理的幂等的

