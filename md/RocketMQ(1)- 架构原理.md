> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/qdhxhz/p/11094624.html)

RocketMQ(1)- 架构原理


=====================

RocketMQ 是阿里开源的分布式消息中间件，跟其它中间件相比，RocketMQ 的特点是**纯 JAVA 实现**；**集群和 HA 实现相对简单**；**在发生宕机和其它故障时消息丢失率更低**。

一、RocketMQ 专业术语
---------------

先讲专业术语的含义，后面会画流程图来更好的去理解它们。

#### Producer

消息生产者，位于用户的进程内，`Producer通过NameServer获取所有Broker的路由信息`，根据负载均衡策略选择将消息发到哪个 Broker，然后调用 Broker 接口提交消息。

#### Producer Group

生产者组，简单来说就是多个发送同一类消息的生产者称之为一个生产者组。

#### Consumer

消息消费者，位于用户进程内。Consumer 通过 NameServer 获取所有 broker 的路由信息后，向 Broker 发送 Pull 请求来获取消息数据。Consumer 可以以两种模式启动，**广播（Broadcast）和集群（Cluster）**，**广播模式下，一条消息会发送给所有 Consumer，集群模式下消息只会发送给一个 Consumer**。

#### Consumer Group

消费者组，和生产者类似，消费同一类消息的多个 Consumer 实例组成一个消费者组。

#### Topic

Topic 用于将消息按主题做划分，**Producer 将消息发往指定的 Topic，Consumer 订阅该 Topic 就可以收到这条消息**。Topic 跟发送方和消费方都没有强关联关系，发送方可以同时往多个 Topic 投放消息，消费方也可以订阅多个 Topic 的消息。在 RocketMQ 中，**Topic 是一个上逻辑概念。消息存储不会按 Topic 分开**。

#### Message

代表一条消息，使用`MessageId`唯一识别，用户在发送时可以设置 messageKey，便于之后查询和跟踪。一个 Message 必须指定 Topic，相当于寄信的地址。Message 还有一个可选的 Tag 设置，以便消费端可以基于 Tag 进行过滤消息。也可以添加额外的键值对，例如你需要一个业务 key 来查找 Broker 上的消息，方便在开发过程中诊断问题。

#### Tag

标签可以被认为是对 Topic 进一步细化。一般在相同业务模块中通过引入标签来标记不同用途的消息。

#### Broker

Broker 是 RocketMQ 的核心模块，`负责接收并存储消息`，同时提供 Push/Pull 接口来将消息发送给 Consumer。Consumer 可选择从 Master 或者 Slave 读取数据。多个主 / 从组成 Broker 集群，集群内的 Master 节点之间不做数据交互。Broker 同时提供消息查询的功能，可以通过 MessageID 和 MessageKey 来查询消息。Borker 会将自己的 Topic 配置信息实时同步到 NameServer。

#### Queue

**Topic 和 Queue 是 1 对多的关系**，**一个 Topic 下可以包含多个 Queue**，主要用于负载均衡。发送消息时，用户只指定 Topic，Producer 会根据 Topic 的路由信息选择具体发到哪个 Queue 上。Consumer 订阅消息时，会根据负载均衡策略决定订阅哪些 Queue 的消息。

#### Offset

RocketMQ 在存储消息时会为每个 Topic 下的每个 Queue 生成一个消息的索引文件，每个 Queue 都对应一个 Offset **记录当前 Queue 中消息条数**。

#### NameServer

NameServer 可以看作是 RocketMQ 的注册中心，它管理两部分数据：集群的 Topic-Queue 的路由配置；Broker 的实时配置信息。其它模块通过 Nameserv 提供的接口获取最新的 Topic 配置和路由信息。

*   `Producer/Consumer` ：通过查询接口获取 Topic 对应的 Broker 的地址信息
*   `Broker` ： 注册配置信息到 NameServer， 实时更新 Topic 信息到 NameServer

二、流程图
-----

我们由简单到复杂的来理解，它的一些核心概念  
![](https://img2018.cnblogs.com/blog/1090617/201906/1090617-20190626173010056-1457807155.jpg)

这个图很好理解，消息先发到 Topic，然后消费者去 Topic 拿消息。只是 Topic 在这里只是个概念，那它到底是怎么存储消息数据的呢，这里就要引入 Broker 概念。

#### 2、Topic 的存储

​ Topic 是一个逻辑上的概念，实际上 Message 是在每个 Broker 上以 Queue 的形式记录。  
![](https://img2018.cnblogs.com/blog/1090617/201906/1090617-20190626173042073-147043337.jpg)

从上面的图片可以总结下几条结论。

```
1、消费者发送的Message会在Broker中的Queue队列中记录。
2、一个Topic的数据可能会存在多个Broker中。
3、一个Broker存在多个Queue。
4、单个的Queue也可能存储多个Topic的消息。
```

也就是说每个 Topic 在 Broker 上会划分成几个逻辑队列，每个逻辑队列保存一部分消息数据，但是保存的消息数据实际上不是真正的消息数据，而是指向 commit log 的消息索引。

`Queue不是真正存储Message的地方，真正存储Message的地方是在CommitLog`。

如图（盗图）  
![](https://img2018.cnblogs.com/blog/1090617/201906/1090617-20190626235211016-2054524747.png)

左边的是 CommitLog。这个是真正存储消息的地方。RocketMQ 所有生产者的消息都是往这一个地方存的。

右边是 ConsumeQueue。这是一个逻辑队列。和上文中 Topic 下的 Queue 是一一对应的。消费者是直接和 ConsumeQueue 打交道。ConsumeQueue 记录了消费位点，这个消费位点关联了 commitlog 的位置。所以即使 ConsumeQueue 出问题，只要 commitlog 还在，消息就没丢，可以恢复出来。还可以通过修改消费位点来重放或跳过一些消息。

#### 3、部署模型

在部署 RocketMQ 时，会部署两种角色。NameServer 和 Broker。如图（盗图）  
![](https://img2018.cnblogs.com/blog/1090617/201906/1090617-20190626233829426-1023022108.png)

针对这张图做个说明

```
1、Product和consumer集群部署，是你开发的项目进行集群部署。
2、Broker 集群部署是为了高可用，因为Broker是真正存储Message的地方，集群部署是为了避免一台挂掉，导致整个项目KO.
```

那 Name SerVer 是做什么用呢，它和 Product、Consumer、Broker 之前存在怎样的关系呢？

先简单概括 Name Server 的特点

```
1、Name Server是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
2、每个Broker与Name Server集群中的所有节点建立长连接，定时注册Topic信息到所有Name Server。
3、Producer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息。
4、Consumer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息。
```

这里面最核心的是`每个Broker与Name Server集群中的所有节点建立长连接`这样做好处多多。

1、这样可以使 Name Server 之间可以没有任何关联，因为它们绑定的 Broker 是一致的。

2、作为 Producer 或者 Consumer 可以绑定任何一个 Name Server 因为它们都是一样的。

三、详解 Broker
-----------

#### 1、Broker 与 Name Server 关系

**1）连接** 单个 Broker 和所有 Name Server 保持长连接。

**2）心跳**

**心跳间隔**：每隔 **30 秒**向所有 NameServer 发送心跳，心跳包含了自身的 Topic 配置信息。

**心跳超时**：NameServer 每隔 **10 秒**，扫描所有还存活的 Broker 连接，若某个连接 2 分钟内没有发送心跳数据，则断开连接。

**3）断开**：当 Broker 挂掉；NameServer 会根据心跳超时主动关闭连接, 一旦连接断开，会更新 Topic 与队列的对应关系，但不会通知生产者和消费者。

#### 2、 负载均衡

一个 Topic 分布在多个 Broker 上，一个 Broker 可以配置多个 Topic，它们是多对多的关系。  
如果某个 Topic 消息量很大，应该给它多配置几个 Queue，并且尽量多分布在不同 Broker 上，减轻某个 Broker 的压力。

#### 3 、可用性

由于消息分布在各个 Broker 上，一旦某个 Broker 宕机，则该 Broker 上的消息读写都会受到影响。

所以 RocketMQ 提供了 Master/Slave 的结构，Salve 定时从 Master 同步数据，如果 Master 宕机，则 Slave 提供消费服务，但是不能写入消息，此过程对应用透明，由 RocketMQ 内部解决。  
有两个关键点：  
`思考1`一旦某个 broker master 宕机，生产者和消费者多久才能发现？

受限于 Rocketmq 的网络连接机制，默认情况下最多需要 **30 秒**，因为消费者每隔 30 秒从 nameserver 获取所有 topic 的最新队列情况，这意味着某个 broker 如果宕机，客户端最多要 30 秒才能感知。

`思考2` master 恢复恢复后，消息能否恢复。  
消费者得到 Master 宕机通知后，转向 Slave 消费，但是 Slave 不能保证 Master 的消息 100% 都同步过来了，因此会有少量的消息丢失。但是消息最终不会丢的，一旦 Master 恢复，未同步过去的消息会被消费掉。

四 Consumer (消费者)
----------------

#### 1 、Consumer 与 Name Server 关系

**1）连接** : 单个 Consumer 和一台 NameServer 保持长连接，如果该 NameServer 挂掉，消费者会自动连接下一个 NameServer，直到有可用连接为止，并能自动重连。  
**2）心跳**: 与 NameServer 没有心跳  
**3）轮询时间** : 默认情况下，消费者每隔 **30 秒**从 NameServer 获取所有 Topic 的最新队列情况，这意味着某个 Broker 如果宕机，客户端最多要 30 秒才能感知。

#### 2、 Consumer 与 Broker 关系

**1）连接** : 单个消费者和该消费者关联的所有 broker 保持长连接。

#### 3、 负载均衡

集群消费模式下，一个消费者集群多台机器共同消费一个 Topic 的多个队列，一个队列只会被一个消费者消费。如果某个消费者挂掉，分组内其它消费者会接替挂掉的消费者继续消费。

五、 Producer(生产者)
----------------

#### 1、 Producer 与 Name Server 关系

**1）连接** 单个 Producer 和一台 NameServer 保持长连接，如果该 NameServer 挂掉，生产者会自动连接下一个 NameServer，直到有可用连接为止，并能自动重连。  
**2）轮询时间** 默认情况下，生产者每隔 30 秒从 NameServer 获取所有 Topic 的最新队列情况，这意味着某个 Broker 如果宕机，生产者最多要 30 秒才能感知，在此期间，  
发往该 broker 的消息发送失败。  
**3）心跳** 与 nameserver 没有心跳

#### 2、 与 broker 关系

**连接** 单个生产者和该生产者关联的所有 broker 保持长连接。

### 参考

[1、十分钟入门 RocketMQ](http://jm.taobao.org/2017/01/12/rocketmq-quick-start-in-10-minutes/)

[2、RocketMQ nameserver、broker 之间的关系](https://blog.csdn.net/linyaogai/article/details/77876078)

[3、RocketMQ-NameServer](https://www.jianshu.com/p/3d8d594d9161)

```
只要自己变优秀了，其他的事情才会跟着好起来（中将8）
```