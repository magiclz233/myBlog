> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/dJz63WQl7UDjcbmDy06FyA)

点击上方 “陶陶技术笔记” 关注我

回复 “资料” 获取作者整理的大量学习资料！

![](https://mmbiz.qpic.cn/mmbiz_jpg/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXyIoooiaDSBiaJhRHSe0xgiclGtcOuszmDGYYYMBDnfeHU7npKRM2Vib5pw/640?wx_fmt=jpeg)

一、背景
----

在微服务架构中，我们常常使用异步化的手段来提升系统的 吞吐量 和 解耦 上下游，而构建异步架构最常用的手段就是使用 `消息队列(MQ)`，那异步架构怎样才能实现数据一致性呢？本文主要介绍如何使用`RocketMQ`的`事务消息`来解决一致性问题。

> RocketMQ 是阿里巴巴开源的分布式消息中间件，目前已成为 Apache 的顶级项目。历经多次天猫双十一海量消息考验，具有高性能、低延时和高可靠等特性

PS：同步场景怎样保证一致性？请看文章《[Spring Cloud 同步场景分布式事务怎样做？试试 Seata](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483735&idx=1&sn=ac2f5d3bf82254bc5f1d3200119b4c32&scene=21#wechat_redirect)》

二、MQ 选型
-------

可以看到在 业务处理 方面来说 `RocketMQ` 优于其他对手，而且原生支持 事务消息

![](https://mmbiz.qpic.cn/mmbiz_jpg/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXMZmyJOY6ticADbQxSBskJYH0Z6bsnpepQGDymbJBBTRKoahZwnUEVHA/640?wx_fmt=jpeg)

PS：业务系统用的是其他 `MQ` 产品但是又需要 事务消息 怎么办？学习原理自己开发实现！

三、什么是事务消息
---------

例如下图的场景：生成订单记录 -> MQ -> 增加积分

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXc3PdEsCh6Htxlic8C4bptVLZlDRxRkAicEqRAo5KTf9Ayt24bbkPTlzg/640?wx_fmt=png)

我们是应该先 创建订单记录，还是先 发送 MQ 消息 呢？

1.  先发送 MQ 消息：这个明显是不行的，因为如果消息发送成功，而订单创建失败的话是没办法把消息收回来的
    
2.  先创建订单记录：如果订单创建成功后 MQ 消息发送失败 抛出异常，因为两个操作都在本地事务中所以订单数据是可以 回滚 的
    

上面的 方式二 看似没问题，但是 网络是不可靠的！如果 `MQ` 的响应因为网络原因没有收到，所以在面对不确定的结果只好进行回滚；但是 `MQ` 端又确实是收到了这条消息的，只是回给客户端的 响应丢失 了！  
   
所以 `事务消息` 就是用来保证 本地事务 与 MQ 消息发送 的原子性！

四、RocketMQ 事务消息原理
-----------------

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXxNtgUC3HIRr3WWYP0RZibVmlr7HOeNpGbQVtxfsB07Q9qsHW9G4VeNQ/640?wx_fmt=png)

主要的逻辑分为两个流程：

*   事务消息发送及提交：
    

1.  发送 `half消息`
    
2.  `MQ服务端` 响应消息写入结果
    
3.  根据发送结果执行 `本地事务`（如果写入失败，此时 half 消息对业务 不可见，本地逻辑不执行）
    
4.  根据本地事务状态执行 `Commit` 或者 `Rollback`（Commit 操作生成消息索引，消息对消费者 可见）  
     
    

*   回查流程：
    

1.  对于长时间没有 `Commit/Rollback` 的事务消息（`pending` 状态的消息），从服务端发起一次 回查
    
2.  `Producer` 收到回查消息，检查回查消息对应的 `本地事务状态`
    
3.  根据本地事务状态，重新 `Commit` 或者 `Rollback`
    

   
逻辑时序图

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXd3A3Vl2gEeOianicUj6ehlkUqpDyRnS9MtWtDibOCg1oTaicsA5vZtA4Xg/640?wx_fmt=png)

五、异步架构一致性实现思路
-------------

从上面的原理可以发现 `事务消息` 仅仅只是保证本地事务和 MQ 消息发送形成整体的 `原子性`，而投递到 MQ 服务器后，并无法保证消费者一定能消费成功！  
   
如果 消费端消费失败 后的处理方式，建议是记录异常信息然后 人工处理，并不建议回滚上游服务的数据 (因为两者是 解耦 的，而且 复杂度 太高)  
   
我们可以利用 `MQ` 的两个特性 `重试` 和 `死信队列` 来协助消费端处理：

1.  消费失败后进行一定次数的 `重试`
    
2.  重试后也失败的话该消息丢进 `死信队列` 里
    
3.  另外起一个线程监听消费 `死信队列` 里的消息，记录日志并且预警！
    

因为有 `重试` 所以消费者需要实现`幂等性`

六、分布式事务场景样例
-----------

下面就用刚刚提到的场景：**生成订单记录 -> MQ -> 增加积分**；来简单讲一下 `Spring Cloud` 中应该怎么做，详细代码请 下载 demo 查看。  
PS：怎样安装部署 RocketMQ 可以参考《[Apache RocketMQ 消息队列部署与可视化界面安装](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483743&idx=1&sn=ed33ae9d289fcd255a7ecb3bf42f0324&scene=21#wechat_redirect)》

### 6.1. 引入依赖

使用 `spring-cloud-stream` 框架来访问 `RocketMQ`

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicYttRVRp9yyftfeBiaPoAQC906gib2pv68uYXpOzBk5dr5jKW7ehoaxS7uqQjEsnFPtyEj3JuIGUf3A/640?wx_fmt=png)

> Spring Cloud Stream 是一个构建消息驱动的框架，通过抽象的定义实现应用与 MQ 消息队列之间的解耦，目前支持 `RabbitMQ`、`kafka` 和 `RocketMQ`  
> ![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicYttRVRp9yyftfeBiaPoAQC9kbqfnNGDeVPH6KiaBjiaPwI01ThHJcyicUx1dSiaiaLP5EQqiclMsBkzeBjw/640?wx_fmt=png)

### 6.2. 开启事务消息

消息生产者需要添加 `transactional: true` 开启 事务消息

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXX39AciaxkGf4WNaoH1QWsytdJ8RFDPU3YXvFsxicmCxd0ZvbGDh25L0kg/640?wx_fmt=png)

### 6.3. 订单服务发送 half 消息

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXZ1JcGAcakVFF85NLpyZHrXOibDZCo2kOT7sM5SBKU4uYI4gFRQSCm0g/640?wx_fmt=png)

> 因为开启了 `事务消息` 所以这里发送的是 `half消息` 对于消费端是 `不可见` 的

### 6.4. 订单服务监听 half 消息

使用 `@RocketMQTransactionListener` 注解监听 半消息，并实现 `RocketMQLocalTransactionListener` 接口，该接口有两个方法

*   executeLocalTransaction：用于提交本地事务
    
*   checkLocalTransaction：用于事务回查
    

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXX6latLRtxtJKibTiaFe7EcWlx7AUNF6RrXTjK3u4PtbXmG7d6YAAribIRw/640?wx_fmt=png)

> 如果提交事务消息失败，需等待约 1 分钟左右 事务回查 方法才会被调用

### 6.5. 积分服务消费消息

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXXAQJt0uOwcS8QcNanUKO47GghLceGL7ibg9FSF3MOtYFy1mkRH2PFHdQ/640?wx_fmt=png)

> 注意：因为有 重试 这里如果是真实的业务需要自行实现 `幂等性`

### 6.6. 消费死信队列预警

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicb0ribNrNjtZ8GxPSEdStJXX4qWsTaiaN9GVgU4jj8xxMQnO1P8Zn3dCOu2kB0RUsGp3MVEjiaficWFZw/640?wx_fmt=png)

> 监听并消费死信队列中的消息，用于记录错误日志，并且预警通知运维人员等

### 6.7. 测试用例

demo 中提供了 3 个接口分别测试不同的场景：

*   事务成功  
    http://localhost:11002/success  
    流程如下：
    

1.  订单创建 **成功**
    
2.  提交事务消息 **成功**
    
3.  消费消息增加积分 **成功**
    

*   订单创建成功但提交事务消息失败  
    http://localhost:11002/produceError  
    流程如下：
    

1.  订单创建 **成功**
    
2.  提交事务消息 **失败**
    
3.  事务回查 (等待 1 分钟左右) **成功**
    
4.  提交事务消息 **成功**
    
5.  消费消息增加积分 **成功**
    

*   消费消息失败  
    http://localhost:11002/consumeError  
    流程如下：
    

1.  订单创建 **成功**
    
2.  提交事务消息 **成功**
    
3.  消费消息增加积分 **失败**
    
4.  重试消费消息 **失败**
    
5.  进入死信队列 **成功**
    
6.  消费死信队列的消息 **成功**
    
7.  记录日志并发出预警 **成功**
    

七、demo 下载地址
-----------

https://gitee.com/zlt2000/microservices-platform/tree/master/zlt-demo/rocketmq-demo/rocketmq-transactional

往期精彩回顾

*   [日志排查问题困难？分布式日志链路跟踪来帮你](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483653&idx=1&sn=df9bf5e428c6908e2fee3f4b32c2a918&scene=21#wechat_redirect)  
    
*   [](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483694&idx=1&sn=4bd9c5865f18f25dd3f38e1f0a9e8701&chksm=a73f686f9048e17903787b98dda5d5c74bb165c27b44a44887e23b6f534d3a54c3b6a13f1d3b&token=19543220&lang=zh_CN&scene=21#wechat_redirect)[Spring Cloud Zuul 的动态路由怎样做？集成 Nacos 实现很简单](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483694&idx=1&sn=4bd9c5865f18f25dd3f38e1f0a9e8701&scene=21#wechat_redirect)  
    
*   [限流怎么做？网关 zuul 集成 Sentinel 最新的网关流控组件](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483674&idx=1&sn=ca1148576652ac130501207a1898e1c2&scene=21#wechat_redirect)
    
*   [为什么我的 Spring Boot 自定义配置项在 IDE 里面不会自动提示？](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483678&idx=1&sn=39f02ea3617f2d46dd005568d9b7ed0d&chksm=a73f685f9048e14951f626c415f5a7c8a8f1ee82c3d9411b52a44f8013d107c3001e4468c49d&token=2075776326&lang=zh_CN&scene=21#wechat_redirect)
    
*   [阿里注册中心 Nacos 生产部署方案](http://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483680&idx=1&sn=f9767a78827e5cc189e97c03054bdf15&chksm=a73f68619048e1775d7eca5531b565ca36ed1d122ba07f8d33b4353918863470ba24be79daa5&scene=21#wechat_redirect)
    
*   [Spring Cloud 开发人员如何解决服务冲突和实例乱窜？](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483710&idx=1&sn=1a75206924bb4f94679318da2a61db1a&scene=21#wechat_redirect)
    
*   [Spring Cloud 同步场景分布式事务怎样做？试试 Seata](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483735&idx=1&sn=ac2f5d3bf82254bc5f1d3200119b4c32&scene=21#wechat_redirect)[](https://mp.weixin.qq.com/s?__biz=MjM5OTI2NDMwMg==&mid=2247483735&idx=1&sn=ac2f5d3bf82254bc5f1d3200119b4c32&scene=21#wechat_redirect)  
    

![](https://mmbiz.qpic.cn/mmbiz_png/Hic4icvNgYCicZAj8c0LR29sqZolaQO4QNYSTOFg9dTfnGQy4bLlib8NAsMZdhC4r4A951ibpnHKiafCDZvo7amHP3CQ/640?wx_fmt=png)

我就知道你 “在看”

![](https://mmbiz.qpic.cn/mmbiz_gif/Hic4icvNgYCicYTna0VAbibCWLAS5QNEZT4quNBEGlbGvhjcUXhHTEuGc1Lic8tLYdypMcH0AKeiakab9efd6ZPQV1hw/640?wx_fmt=gif)