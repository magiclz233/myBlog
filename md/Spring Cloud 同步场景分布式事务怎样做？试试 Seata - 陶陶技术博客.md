> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zlt2000.gitee.io](https://zlt2000.gitee.io/2019-09-16-spring-cloud-seata/)

[![](http://qiniu.zlt2000.cn/blog/20200417/g9F3GiArgo4H.png?imageslim)](https://zlt2000.gitee.io/2019-09-16-spring-cloud-seata/ "Spring Cloud同步场景分布式事务怎样做？试试Seata")

[](#一、概述 "一、概述")一、概述
--------------------

在微服务架构下，虽然我们会尽量避免分布式事务，但是只要业务复杂的情况下这是一个绕不开的问题，如何保证业务数据一致性呢？本文主要介绍同步场景下使用`Seata`的`AT模式`来解决一致性问题。

> `Seata`是 **阿里巴巴** 开源的 **一站式分布式事务解决方案** 中间件，以 **高效** 并且对业务 **0 侵入** 的方式，解决 **微服务** 场景下面临的分布式事务问题

[](#二、Seata介绍 "二、Seata介绍")二、Seata 介绍
------------------------------------

整体事务逻辑是基于 **两阶段提交** 的模型，核心概念包括以下 3 个角色：

*   **TM**：事务的发起者。用来告诉 TC，全局事务的开始，提交，回滚。
*   **RM**：具体的事务资源，每一个 RM 都会作为一个分支事务注册在 TC。
*   **TC**：事务的协调者 seata-server，用于接收我们的事务的注册，提交和回滚。

目前的`Seata`有两种模式可使用分别对应不同业务场景

### [](#2-1-AT模式 "2.1. AT模式")2.1. AT 模式

**该模式适合的场景：**

*   基于支持本地 `ACID` 事务的关系型数据库。
*   Java 应用，通过 `JDBC` 访问数据库。

![](http://qiniu.zlt2000.cn/blog/20190918/2Xv2Ska1Kdn4.jpg?imageslim)

**一个典型的分布式事务过程：**

1.  `TM` 向 `TC` 申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的 `XID`。
2.  `XID` 在微服务调用链路的上下文中传播。
3.  `RM` 向 `TC` 注册分支事务，将其纳入 XID 对应全局事务的管辖。
4.  `TM` 向 `TC` 发起针对 `XID` 的全局提交或回滚决议。
5.  `TC` 调度 `XID` 下管辖的全部分支事务完成提交或回滚请求。

### [](#2-2-MT模式 "2.2. MT模式")2.2. MT 模式

该模式逻辑类似`TCC`，需要 **自定义实现** `prepare`、`commit`和`rollback`的逻辑，适合 **非关系型数据库** 的场景  
![](http://qiniu.zlt2000.cn/blog/20190918/m7slPzL2O8he.png?imageslim)

[](#三、Seata场景样例 "三、Seata场景样例")三、Seata 场景样例
------------------------------------------

模拟一个简单的用户下单场景，4 个子工程分别是 **Bussiness(事务发起者)**、**Order(创建订单)**、**Storage(扣减库存)** 和 **Account(扣减账户余额)**  
![](http://qiniu.zlt2000.cn/blog/20190918/t4DVUlQ8AbXp.png?imageslim)

### [](#3-1-部署Seata的Server端 "3.1. 部署Seata的Server端")3.1. 部署 Seata 的 Server 端

![](http://qiniu.zlt2000.cn/blog/20190918/ICDASifbfxK0.jpg?imageslim)  
`Discover`注册、`Config`配置和`Store`存储模块默认都是使用`file`只能适用于单机，我们安装的时候分别改成使用`nacos`和`Mysql`以支持 server 端集群

#### [](#3-1-1-下载最新版本并解压 "3.1.1. 下载最新版本并解压")3.1.1. 下载最新版本并解压

[https://github.com/seata/seata/releases](https://github.com/seata/seata/releases)

#### [](#3-1-2-修改-conf-registry-conf-配置 "3.1.2. 修改 conf/registry.conf 配置")3.1.2. 修改 conf/registry.conf 配置

注册中心和配置中心默认是`file`这里改为`nacos`；设置 **registry** 和 **config** 节点中的`type`为`nacos`，修改`serverAddr`为你的`nacos`节点地址。

```
registry {
  type = "nacos"

  nacos {
    serverAddr = "192.168.28.130"
    namespace = "public"
    cluster = "default"
  }
}

config {
  type = "nacos"

  nacos {
    serverAddr = "192.168.28.130"
    namespace = "public"
    cluster = "default"
  }
}
```

#### [](#3-1-3-修改-conf-nacos-config-txt配置 "3.1.3. 修改 conf/nacos-config.txt配置")3.1.3. 修改 conf/nacos-config.txt 配置

![](http://qiniu.zlt2000.cn/blog/20190918/4NnH3AlU0kII.png?imageslim)

*   修改 **service.vgroup_mapping** 为自己应用对应的名称；如果有多个服务，添加相应的配置
    
    > 默认组名为`${spring.application.name}-fescar-service-group`，可通过`spring.cloud.alibaba.seata.tx-service-group`配置修改
    
*   修改 **store.mode** 为`db`，并修改数据库相关配置
    

#### [](#3-1-4-初始化seata的nacos配置 "3.1.4. 初始化seata的nacos配置")3.1.4. 初始化 seata 的 nacos 配置

```
cd conf
sh nacos-config.sh 192.168.28.130
```

成功后在`nacos`的配置列表中能看到`seata`的相关配置

![](http://qiniu.zlt2000.cn/blog/20190918/riHOrE2mOGXW.png?imageslim)

#### [](#3-1-5-初始化数据库 "3.1.5. 初始化数据库")3.1.5. 初始化数据库

执行`conf/db_store.sql`中的脚本

#### [](#3-1-6-启动seata-server "3.1.6. 启动seata-server")3.1.6. 启动 seata-server

```
sh bin/seata-server.sh -p 8091 -h 192.168.28.130
```

### [](#3-2-应用配置 "3.2. 应用配置")3.2. 应用配置

#### [](#3-2-1-初始化数据库 "3.2.1. 初始化数据库")3.2.1. 初始化数据库

执行脚本 [seata-demo.sql](https://gitee.com/zlt2000/microservices-platform/blob/master/zlt-demo/seata-demo/seata-demo.sql)

> 需在业务相关的数据库中添加 **undo_log** 表，用于保存需要回滚的数据

#### [](#3-2-2-添加registry-conf配置 "3.2.2. 添加registry.conf配置")3.2.2. 添加 registry.conf 配置

直接把 **seata-server** 中的`registry.conf`复制到每个服务中去即可，不需要修改  
![](http://qiniu.zlt2000.cn/blog/20190918/1W1lrxvaioJO.png?imageslim)

#### [](#3-2-3-修改配置 "3.2.3. 修改配置")3.2.3. 修改配置

demo 中的每个服务各自修改配置文件

*   **bootstrap.yml** 修改 nacos 地址
*   **application.yml** 修改数据库配置

#### [](#3-2-4-配置数据源代理 "3.2.4. 配置数据源代理")3.2.4. 配置数据源代理

`Seata`是通过代理数据源实现分布式事务，所以需要配置`io.seata.rm.datasource.DataSourceProxy`的`Bean`，且是`@Primary`默认的数据源，否则事务不会回滚，无法实现分布式事务

```
public class DataSourceProxyConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DruidDataSource druidDataSource() {
        return new DruidDataSource();
    }

    @Primary
    @Bean
    public DataSourceProxy dataSourceProxy(DruidDataSource druidDataSource) {
        return new DataSourceProxy(druidDataSource);
    }
}
```

因为使用了 mybatis 的 starter 所以需要排除`DataSourceAutoConfiguration`，不然会产生循环依赖

```
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

#### [](#3-2-5-事务发起者添加全局事务注解 "3.2.5. 事务发起者添加全局事务注解")3.2.5. 事务发起者添加全局事务注解

事务发起者 `business-service` 添加 `@GlobalTransactional` 注解

```
@GlobalTransactional
public void placeOrder(String userId) {
    ......
}
```

### [](#3-3-测试 "3.3. 测试")3.3. 测试

提供两个接口测试

1.  事务成功：扣除库存成功 > 创建订单成功 > 扣减账户余额成功  
    [http://localhost:9090/placeOrder](http://localhost:9090/placeOrder)
2.  事务失败：扣除库存成功 > 创建订单成功 > 扣减账户余额失败，事务回滚  
    [http://localhost:9090/placeOrderFallBack](http://localhost:9090/placeOrderFallBack)

### [](#3-4-demo下载地址 "3.4. demo下载地址")3.4. demo 下载地址

[https://gitee.com/zlt2000/microservices-platform/tree/master/zlt-demo/seata-demo](https://gitee.com/zlt2000/microservices-platform/tree/master/zlt-demo/seata-demo)

  

[![](http://qiniu.zlt2000.cn/blog/20190902/M56cWjw7uNsc.png?imageslim)](http://qiniu.zlt2000.cn/blog/20190902/M56cWjw7uNsc.png?imageslim)