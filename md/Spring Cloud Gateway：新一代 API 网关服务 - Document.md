> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.macrozheng.com](http://www.macrozheng.com/#/cloud/gateway)

学习不走弯路，[关注公众号](#/cloud/gateway?id=%e5%85%ac%e4%bc%97%e5%8f%b7) 回复「学习路线」，获取 mall 项目专属学习路线！

> Spring Cloud Gateway 为 SpringBoot 应用提供了 API 网关支持，具有强大的智能路由与过滤器功能，本文将对其用法进行详细介绍。

[Gateway 简介](#/cloud/gateway?id=gateway-%e7%ae%80%e4%bb%8b)
-----------------------------------------------------------

Gateway 是在 Spring 生态系统之上构建的 API 网关服务，基于 Spring 5，Spring Boot 2 和 Project Reactor 等技术。Gateway 旨在提供一种简单而有效的方式来对 API 进行路由，以及提供一些强大的过滤器功能， 例如：熔断、限流、重试等。

Spring Cloud Gateway 具有如下特性：

*   基于 Spring Framework 5, Project Reactor 和 Spring Boot 2.0 进行构建；
*   动态路由：能够匹配任何请求属性；
*   可以对路由指定 Predicate（断言）和 Filter（过滤器）；
*   集成 Hystrix 的断路器功能；
*   集成 Spring Cloud 服务发现功能；
*   易于编写的 Predicate（断言）和 Filter（过滤器）；
*   请求限流功能；
*   支持路径重写。

[相关概念](#/cloud/gateway?id=%e7%9b%b8%e5%85%b3%e6%a6%82%e5%bf%b5)
---------------------------------------------------------------

*   Route（路由）：路由是构建网关的基本模块，它由 ID，目标 URI，一系列的断言和过滤器组成，如果断言为 true 则匹配该路由；
*   Predicate（断言）：指的是 Java 8 的 Function Predicate。 输入类型是 Spring 框架中的 ServerWebExchange。 这使开发人员可以匹配 HTTP 请求中的所有内容，例如请求头或请求参数。如果请求与断言相匹配，则进行路由；
*   Filter（过滤器）：指的是 Spring 框架中 GatewayFilter 的实例，使用过滤器，可以在请求被路由前后对请求进行修改。

[创建 api-gateway 模块](#/cloud/gateway?id=%e5%88%9b%e5%bb%ba-api-gateway%e6%a8%a1%e5%9d%97)
----------------------------------------------------------------------------------------

> 这里我们创建一个 api-gateway 模块来演示 Gateway 的常用功能。

### [在 pom.xml 中添加相关依赖](#/cloud/gateway?id=%e5%9c%a8pomxml%e4%b8%ad%e6%b7%bb%e5%8a%a0%e7%9b%b8%e5%85%b3%e4%be%9d%e8%b5%96)

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### [两种不同的配置路由方式](#/cloud/gateway?id=%e4%b8%a4%e7%a7%8d%e4%b8%8d%e5%90%8c%e7%9a%84%e9%85%8d%e7%bd%ae%e8%b7%af%e7%94%b1%e6%96%b9%e5%bc%8f)

> Gateway 提供了两种不同的方式用于配置路由，一种是通过 yml 文件来配置，另一种是通过 Java Bean 来配置，下面我们分别介绍下。

#### [使用 yml 配置](#/cloud/gateway?id=%e4%bd%bf%e7%94%a8yml%e9%85%8d%e7%bd%ae)

*   在 application.yml 中进行配置：

```
server:
  port: 9201
service-url:
  user-service: http://localhost:8201
spring:
  cloud:
    gateway:
      routes:
        - id: path_route #路由的ID
          uri: ${service-url.user-service}/user/{id} #匹配后路由地址
          predicates: # 断言，路径相匹配的进行路由
            - Path=/user/{id}
```

*   启动 eureka-server，user-service 和 api-gateway 服务，并调用该地址测试：[http://localhost:9201/user/1](http://localhost:9201/user/1)
    
*   我们发现该请求被路由到了 user-service 的该路径上：[http://localhost:8201/user/1](http://localhost:8201/user/1)
    

![](http://www.macrozheng.com/images/springcloud_gateway_01.png)

#### [使用 Java Bean 配置](#/cloud/gateway?id=%e4%bd%bf%e7%94%a8java-bean%e9%85%8d%e7%bd%ae)

*   添加相关配置类，并配置一个 RouteLocator 对象：

```
/**
 * Created by macro on 2019/9/24.
 */
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("path_route2", r -> r.path("/user/getByUsername")
                        .uri("http://localhost:8201/user/getByUsername"))
                .build();
    }
}
```

*   重新启动 api-gateway 服务，并调用该地址测试：[http://localhost:8201/user/getByUsername?username=macro](http://localhost:9201/user/getByUsername?user>http://localhost:9201/user/getByUsername?username=macro</a></p></li><li><p> 我们发现该请求被路由到了 user-service 的该路径上：<a href=)
    

![](http://www.macrozheng.com/images/springcloud_gateway_02.png)

[Route Predicate 的使用](#/cloud/gateway?id=route-predicate-%e7%9a%84%e4%bd%bf%e7%94%a8)
-------------------------------------------------------------------------------------

Spring Cloud Gateway 将路由匹配作为 Spring WebFlux HandlerMapping 基础架构的一部分。 Spring Cloud Gateway 包括许多内置的 Route Predicate 工厂。 所有这些 Predicate 都与 HTTP 请求的不同属性匹配。 多个 Route Predicate 工厂可以进行组合，下面我们来介绍下一些常用的 Route Predicate。

注意：Predicate 中提到的配置都在 application-predicate.yml 文件中进行修改，并用该配置启动 api-gateway 服务。

### [After Route Predicate](#/cloud/gateway?id=after-route-predicate)

在指定时间之后的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: ${service-url.user-service}
          predicates:
            - After=2019-09-24T16:30:00+08:00[Asia/Shanghai]
```

### [Before Route Predicate](#/cloud/gateway?id=before-route-predicate)

在指定时间之前的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: before_route
          uri: ${service-url.user-service}
          predicates:
            - Before=2019-09-24T16:30:00+08:00[Asia/Shanghai]
```

### [Between Route Predicate](#/cloud/gateway?id=between-route-predicate)

在指定时间区间内的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: before_route
          uri: ${service-url.user-service}
          predicates:
            - Between=2019-09-24T16:30:00+08:00[Asia/Shanghai], 2019-09-25T16:30:00+08:00[Asia/Shanghai]
```

### [Cookie Route Predicate](#/cloud/gateway?id=cookie-route-predicate)

带有指定 Cookie 的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: cookie_route
          uri: ${service-url.user-service}
          predicates:
            - Cookie=username,macro
```

使用 curl 工具发送带有 cookie 为`username=macro`的请求可以匹配该路由。

```
curl http://localhost:9201/user/1 --cookie "user
```

带有指定请求头的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: ${service-url.user-service}
        predicates:
        - Header=X-Request-Id, \d+
```

使用 curl 工具发送带有请求头为`X-Request-Id:123`的请求可以匹配该路由。

```
curl http://localhost:9201/user/1 -H "X-Request-Id:123"
```

### [Host Route Predicate](#/cloud/gateway?id=host-route-predicate)

带有指定 Host 的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: host_route
          uri: ${service-url.user-service}
          predicates:
            - Host=**.macrozheng.com
```

使用 curl 工具发送带有请求头为`Host:www.macrozheng.com`的请求可以匹配该路由。

```
curl http://localhost:9201/user/1 -H "Host:www.macrozheng.com"
```

### [Method Route Predicate](#/cloud/gateway?id=method-route-predicate)

发送指定方法的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: ${service-url.user-service}
        predicates:
        - Method=GET
```

使用 curl 工具发送 GET 请求可以匹配该路由。

```
curl http://localhost:9201/user/1
```

使用 curl 工具发送 POST 请求无法匹配该路由。

```
curl -X POST http://localhost:9201/user/1
```

### [Path Route Predicate](#/cloud/gateway?id=path-route-predicate)

发送指定路径的请求会匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: ${service-url.user-service}/user/{id}
          predicates:
            - Path=/user/{id}
```

使用 curl 工具发送`/user/1`路径请求可以匹配该路由。

```
curl http://localhost:9201/user/1
```

使用 curl 工具发送`/abc/1`路径请求无法匹配该路由。

```
curl http://localhost:9201/abc/1
```

### [Query Route Predicate](#/cloud/gateway?id=query-route-predicate)

带指定查询参数的请求可以匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: ${service-url.user-service}/user/getByUsername
        predicates:
        - Query=username
```

使用 curl 工具发送带`username=macro`查询参数的请求可以匹配该路由。

```
curl http://localhost:9201/user/getByUsername?username=macro
```

使用 curl 工具发送带不带查询参数的请求无法匹配该路由。

```
curl http://localhost:9201/user/getByUsername
```

### [RemoteAddr Route Predicate](#/cloud/gateway?id=remoteaddr-route-predicate)

从指定远程地址发起的请求可以匹配该路由。

```
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: ${service-url.user-service}
        predicates:
        - RemoteAddr=192.168.1.1/24
```

使用 curl 工具从 192.168.1.1 发起请求可以匹配该路由。

```
curl http://localhost:9201/user/1
```

### [Weight Route Predicate](#/cloud/gateway?id=weight-route-predicate)

使用权重来路由相应请求，以下表示有 80% 的请求会被路由到 localhost:8201，20% 会被路由到 localhost:8202。

```
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: http://localhost:8201
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: http://localhost:8202
        predicates:
        - Weight=group1, 2
```

[Route Filter 的使用](#/cloud/gateway?id=route-filter-%e7%9a%84%e4%bd%bf%e7%94%a8)
-------------------------------------------------------------------------------

> 路由过滤器可用于修改进入的 HTTP 请求和返回的 HTTP 响应，路由过滤器只能指定路由进行使用。Spring Cloud Gateway 内置了多种路由过滤器，他们都由 GatewayFilter 的工厂类来产生，下面我们介绍下常用路由过滤器的用法。

### [AddRequestParameter GatewayFilter](#/cloud/gateway?id=addrequestparameter-gatewayfilter)

给请求添加参数的过滤器。

```
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter_route
          uri: http://localhost:8201
          filters:
            - AddRequestParameter=username, macro
          predicates:
            - Method=GET
```

以上配置会对 GET 请求添加`username=macro`的请求参数，通过 curl 工具使用以下命令进行测试。

```
curl http://localhost:9201/user/getByUsername
```

相当于发起该请求：

```
curl http://localhost:8201/user/getByUsername?username=macro
```

### [StripPrefix GatewayFilter](#/cloud/gateway?id=stripprefix-gatewayfilter)

对指定数量的路径前缀进行去除的过滤器。

```
spring:
  cloud:
    gateway:
      routes:
      - id: strip_prefix_route
        uri: http://localhost:8201
        predicates:
        - Path=/user-service/**
        filters:
        - StripPrefix=2
```

以上配置会把以`/user-service/`开头的请求的路径去除两位，通过 curl 工具使用以下命令进行测试。

```
curl http://localhost:9201/user-service/a/user/1
```

相当于发起该请求：

```
curl http://localhost:8201/user/1
```

### [PrefixPath GatewayFilter](#/cloud/gateway?id=prefixpath-gatewayfilter)

与 StripPrefix 过滤器恰好相反，会对原有路径进行增加操作的过滤器。

```
spring:
  cloud:
    gateway:
      routes:
      - id: prefix_path_route
        uri: http://localhost:8201
        predicates:
        - Method=GET
        filters:
        - PrefixPath=/user
```

以上配置会对所有 GET 请求添加`/user`路径前缀，通过 curl 工具使用以下命令进行测试。

```
curl http://localhost:9201/1
```

相当于发起该请求：

```
curl http://localhost:8201/user/1
```

### [Hystrix GatewayFilter](#/cloud/gateway?id=hystrix-gatewayfilter)

Hystrix 过滤器允许你将断路器功能添加到网关路由中，使你的服务免受级联故障的影响，并提供服务降级处理。

*   要开启断路器功能，我们需要在 pom.xml 中添加 Hystrix 的相关依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

*   然后添加相关服务降级的处理类：

```
/**
 * Created by macro on 2019/9/25.
 */
@RestController
public class FallbackController {

    @GetMapping("/fallback")
    public Object fallback() {
        Map<String,Object> result = new HashMap<>();
        result.put("data",null);
        result.put("message","Get request fallback!");
        result.put("code",500);
        return result;
    }
}
```

*   在 application-filter.yml 中添加相关配置，当路由出错时会转发到服务降级处理的控制器上：

```
spring:
  cloud:
    gateway:
      routes:
        - id: hystrix_route
          uri: http://localhost:8201
          predicates:
            - Method=GET
          filters:
            - name: Hystrix
              args:
                name: fallbackcmd
                fallbackUri: forward:/fallback
```

*   关闭 user-service，调用该地址进行测试：[http://localhost:9201/user/1](http://localhost:9201/user/1) ，发现已经返回了服务降级的处理信息。

![](http://www.macrozheng.com/images/springcloud_gateway_03.png)

### [RequestRateLimiter GatewayFilter](#/cloud/gateway?id=requestratelimiter-gatewayfilter)

RequestRateLimiter 过滤器可以用于限流，使用 RateLimiter 实现来确定是否允许当前请求继续进行，如果请求太大默认会返回 HTTP 429 - 太多请求状态。

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

*   添加限流策略的配置类，这里有两种策略一种是根据请求参数中的 username 进行限流，另一种是根据访问 IP 进行限流；

```
/**
 * Created by macro on 2019/9/25.
 */
@Configuration
public class RedisRateLimiterConfig {
    @Bean
    KeyResolver userKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("username"));
    }

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
    }
}
```

*   我们使用 Redis 来进行限流，所以需要添加 Redis 和 RequestRateLimiter 的配置，这里对所有的 GET 请求都进行了按 IP 来限流的操作；

```
server:
  port: 9201
spring:
  redis:
    host: localhost
    password: 123456
    port: 6379
  cloud:
    gateway:
      routes:
        - id: requestratelimiter_route
          uri: http://localhost:8201
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 1 #每秒允许处理的请求数量
                redis-rate-limiter.burstCapacity: 2 #每秒最大处理的请求数量
                key-resolver: "#{@ipKeyResolver}" #限流策略，对应策略的Bean
          predicates:
            - Method=GET
logging:
  level:
    org.springframework.cloud.gateway: debug
```

*   多次请求该地址：[http://localhost:9201/user/1](http://localhost:9201/user/1) ，会返回状态码为 429 的错误；

![](http://www.macrozheng.com/images/springcloud_gateway_04.png)

### [Retry GatewayFilter](#/cloud/gateway?id=retry-gatewayfilter)

对路由请求进行重试的过滤器，可以根据路由请求返回的 HTTP 状态码来确定是否进行重试。

*   修改配置文件：

```
spring:
  cloud:
    gateway:
      routes:
      - id: retry_route
        uri: http://localhost:8201
        predicates:
        - Method=GET
        filters:
        - name: Retry
          args:
            retries: 1 #需要进行重试的次数
            statuses: BAD_GATEWAY #返回哪个状态码需要进行重试，返回状态码为5XX进行重试
            backoff:
              firstBackoff: 10ms
              maxBackoff: 50ms
              factor: 2
              basedOnPreviousValue: false
```

*   当调用返回 500 时会进行重试，访问测试地址：[http://localhost:9201/user/111](http://localhost:9201/user/111)
    
*   可以发现 user-service 控制台报错 2 次，说明进行了一次重试。
    

```
2019-10-27 14:08:53.435 ERROR 2280 --- [nio-8201-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.NullPointerException] with root cause

java.lang.NullPointerException: null
    at com.macro.cloud.controller.UserController.getUser(UserController.java:34) ~[classes/:na]
```

[结合注册中心使用](#/cloud/gateway?id=%e7%bb%93%e5%90%88%e6%b3%a8%e5%86%8c%e4%b8%ad%e5%bf%83%e4%bd%bf%e7%94%a8)
-------------------------------------------------------------------------------------------------------

> 我们上次讲到[使用 Zuul 作为网关](https://mp.weixin.qq.com/s/ttL5gm_ZLAQL_uJ4iNtqsQ)结合注册中心进行使用时，默认情况下 Zuul 会根据注册中心注册的服务列表，以服务名为路径创建动态路由，Gateway 同样也实现了该功能。下面我们演示下 Gateway 结合注册中心如何使用默认的动态路由和过滤器。

### [使用动态路由](#/cloud/gateway?id=%e4%bd%bf%e7%94%a8%e5%8a%a8%e6%80%81%e8%b7%af%e7%94%b1)

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

*   添加 application-eureka.yml 配置文件：

```
server:
  port: 9201
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true #开启从注册中心动态创建路由的功能
          lower-case-service-id: true #使用小写服务名，默认是大写
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8001/eureka/
logging:
  level:
    org.springframework.cloud.gateway: debug
```

*   使用 application-eureka.yml 配置文件启动 api-gateway 服务，访问 [http://localhost:9201/user-service/user/1](http://localhost:9201/user-service/user/1) ，可以路由到 user-service 的 [http://localhost:8201/user/1](http://localhost:8201/user/1) 处。

### [使用过滤器](#/cloud/gateway?id=%e4%bd%bf%e7%94%a8%e8%bf%87%e6%bb%a4%e5%99%a8)

> 在结合注册中心使用过滤器的时候，我们需要注意的是 uri 的协议为`lb`，这样才能启用 Gateway 的负载均衡功能。

*   修改 application-eureka.yml 文件，使用了 PrefixPath 过滤器，会为所有 GET 请求路径添加`/user`路径并路由；

```
server:
  port: 9201
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: prefixpath_route
          uri: lb://user-service #此处需要使用lb协议
          predicates:
            - Method=GET
          filters:
            - PrefixPath=/user
      discovery:
        locator:
          enabled: true
eureka:
  client:
    service-url: 
      defaultZone: http://localhost:8001/eureka/
logging:
  level:
    org.springframework.cloud.gateway: debug
```

*   使用 application-eureka.yml 配置文件启动 api-gateway 服务，访问 [http://localhost:9201/1](http://localhost:9201/1) ，可以路由到 user-service 的 [http://localhost:8201/user/1](http://localhost:8201/user/1) 处。

[使用到的模块](#/cloud/gateway?id=%e4%bd%bf%e7%94%a8%e5%88%b0%e7%9a%84%e6%a8%a1%e5%9d%97)
-----------------------------------------------------------------------------------

```
springcloud-learning
├── eureka-server -- eureka注册中心
├── user-service -- 提供User对象CRUD接口的服务
└── api-gateway -- gateway作为网关的测试服务
```

[项目源码地址](#/cloud/gateway?id=%e9%a1%b9%e7%9b%ae%e6%ba%90%e7%a0%81%e5%9c%b0%e5%9d%80)
-----------------------------------------------------------------------------------

[https://github.com/macrozheng/springcloud-learning](https://github.com/macrozheng/springcloud-learning)

[公众号](#/cloud/gateway?id=%e5%85%ac%e4%bc%97%e5%8f%b7)
-----------------------------------------------------

![](http://macro-oss.oss-cn-shenzhen.aliyuncs.com/mall/banner/qrcode_for_macrozheng_258.jpg)

* * *