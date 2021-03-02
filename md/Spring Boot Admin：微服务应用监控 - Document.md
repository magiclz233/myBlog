> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.macrozheng.com](http://www.macrozheng.com/#/cloud/admin)

学习不走弯路，[关注公众号](#/cloud/admin?id=%e5%85%ac%e4%bc%97%e5%8f%b7) 回复「学习路线」，获取 mall 项目专属学习路线！

> Spring Boot Admin 可以对 SpringBoot 应用的各项指标进行监控，可以作为微服务架构中的监控中心来使用，本文将对其用法进行详细介绍。

[Spring Boot Admin 简介](#/cloud/admin?id=spring-boot-admin-%e7%ae%80%e4%bb%8b)
-----------------------------------------------------------------------------

SpringBoot 应用可以通过 Actuator 来暴露应用运行过程中的各项指标，Spring Boot Admin 通过这些指标来监控 SpringBoot 应用，然后通过图形化界面呈现出来。Spring Boot Admin 不仅可以监控单体应用，还可以和 Spring Cloud 的注册中心相结合来监控微服务应用。

Spring Boot Admin 可以提供应用的以下监控信息：

*   监控应用运行过程中的概览信息；
*   度量指标信息，比如 JVM、Tomcat 及进程信息；
*   环境变量信息，比如系统属性、系统环境变量以及应用配置信息；
*   查看所有创建的 Bean 信息；
*   查看应用中的所有配置信息；
*   查看应用运行日志信息；
*   查看 JVM 信息；
*   查看可以访问的 Web 端点；
*   查看 HTTP 跟踪信息。

[创建 admin-server 模块](#/cloud/admin?id=%e5%88%9b%e5%bb%baadmin-server%e6%a8%a1%e5%9d%97)
---------------------------------------------------------------------------------------

> 这里我们创建一个 admin-server 模块来作为监控中心演示其功能。

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
```

*   在 application.yml 中进行配置：

```
spring:
  application:
    name: admin-server
server:
  port: 9301
```

*   在启动类上添加 @EnableAdminServer 来启用 admin-server 功能：

```
@EnableAdminServer
@SpringBootApplication
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }

}
```

[创建 admin-client 模块](#/cloud/admin?id=%e5%88%9b%e5%bb%baadmin-client%e6%a8%a1%e5%9d%97)
---------------------------------------------------------------------------------------

> 这里我们创建一个 admin-client 模块作为客户端注册到 admin-server。

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

*   在 application.yml 中进行配置：

```
spring:
  application:
    name: admin-client
  boot:
    admin:
      client:
        url: http://localhost:9301 #配置admin-server地址
server:
  port: 9305
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
logging:
  file: admin-client.log #添加开启admin的日志监控
```

*   启动 admin-server 和 admin-client 服务。

[监控信息演示](#/cloud/admin?id=%e7%9b%91%e6%8e%a7%e4%bf%a1%e6%81%af%e6%bc%94%e7%a4%ba)
---------------------------------------------------------------------------------

*   访问如下地址打开 Spring Boot Admin 的主页：[http://localhost:9301](http://localhost:9301/)

![](http://www.macrozheng.com/images/springcloud_admin_01.png)

*   点击 wallboard 按钮，选择 admin-client 查看监控信息；
    
*   监控信息概览；
    

![](http://www.macrozheng.com/images/springcloud_admin_02.png)

*   度量指标信息，比如 JVM、Tomcat 及进程信息；

![](http://www.macrozheng.com/images/springcloud_admin_03.png)

*   环境变量信息，比如系统属性、系统环境变量以及应用配置信息；

![](http://www.macrozheng.com/images/springcloud_admin_04.png)

*   查看所有创建的 Bean 信息；

![](http://www.macrozheng.com/images/springcloud_admin_05.png)

*   查看应用中的所有配置信息；

![](http://www.macrozheng.com/images/springcloud_admin_06.png)

*   查看日志信息，需要添加以下配置才能开启；

```
logging:
  file: admin-client.log #添加开启admin的日志监控
```

![](http://www.macrozheng.com/images/springcloud_admin_07.png)

*   查看 JVM 信息；

![](http://www.macrozheng.com/images/springcloud_admin_08.png)

*   查看可以访问的 Web 端点；

![](http://www.macrozheng.com/images/springcloud_admin_09.png)

*   查看 HTTP 跟踪信息；

![](http://www.macrozheng.com/images/springcloud_admin_10.png)

[结合注册中心使用](#/cloud/admin?id=%e7%bb%93%e5%90%88%e6%b3%a8%e5%86%8c%e4%b8%ad%e5%bf%83%e4%bd%bf%e7%94%a8)
-----------------------------------------------------------------------------------------------------

> Spring Boot Admin 结合 Spring Cloud 注册中心使用，只需将 admin-server 和注册中心整合即可，admin-server 会自动从注册中心获取服务列表，然后挨个获取监控信息。这里以 Eureka 注册中心为例来介绍下该功能。

### [修改 admin-server](#/cloud/admin?id=%e4%bf%ae%e6%94%b9admin-server)

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

*   在 application-eureka.yml 中进行配置，只需添加注册中心配置即可：

```
spring:
  application:
    name: admin-server
server:
  port: 9301
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/
```

*   在启动类上添加 @EnableDiscoveryClient 来启用服务注册功能：

```
@EnableDiscoveryClient
@EnableAdminServer
@SpringBootApplication
public class AdminServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }

}
```

### [修改 admin-client](#/cloud/admin?id=%e4%bf%ae%e6%94%b9admin-client)

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

*   在 application-eureka.yml 中进行配置，删除原来的 admin-server 地址配置，添加注册中心配置即可：

```
spring:
  application:
    name: admin-client
server:
  port: 9305
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
logging:
  file: admin-client.log #添加开启admin的日志监控
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/
```

*   在启动类上添加 @EnableDiscoveryClient 来启用服务注册功能：

```
@EnableDiscoveryClient
@SpringBootApplication
public class AdminClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminClientApplication.class, args);
    }

}
```

### [功能演示](#/cloud/admin?id=%e5%8a%9f%e8%83%bd%e6%bc%94%e7%a4%ba)

*   启动 eureka-server，使用 application-eureka.yml 配置启动 admin-server，admin-client；
    
*   查看注册中心发现服务均已注册：[http://localhost:8001/](http://localhost:8001/)
    

![](http://www.macrozheng.com/images/springcloud_admin_11.png)

*   查看 Spring Boot Admin 主页发现可以看到服务信息：[http://localhost:9301](http://localhost:9301/)

![](http://www.macrozheng.com/images/springcloud_admin_12.png)

[添加登录认证](#/cloud/admin?id=%e6%b7%bb%e5%8a%a0%e7%99%bb%e5%bd%95%e8%ae%a4%e8%af%81)
---------------------------------------------------------------------------------

> 我们可以通过给 admin-server 添加 Spring Security 支持来获得登录认证功能。

### [创建 admin-security-server 模块](#/cloud/admin?id=%e5%88%9b%e5%bb%baadmin-security-server%e6%a8%a1%e5%9d%97)

*   在 pom.xml 中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>2.1.5</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

*   在 application.yml 中进行配置，配置登录用户名和密码，忽略 admin-security-server 的监控信息：

```
spring:
  application:
    name: admin-security-server
  security: # 配置登录用户名和密码
    user:
      name: macro
      password: 123456
  boot:  # 不显示admin-security-server的监控信息
    admin:
      discovery:
        ignored-services: ${spring.application.name}
server:
  port: 9301
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8001/eureka/
```

*   对 SpringSecurity 进行配置，以便 admin-client 可以注册：

```
/**
 * Created by macro on 2019/9/30.
 */
@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {
    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                //1.配置所有静态资源和登录页可以公开访问
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
                //2.配置登录和登出路径
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                //3.开启http basic支持，admin-client注册时需要使用
                .httpBasic().and()
                .csrf()
                //4.开启基于cookie的csrf保护
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                //5.忽略这些路径的csrf保护以便admin-client注册
                .ignoringAntMatchers(
                        adminContextPath + "/instances",
                        adminContextPath + "/actuator/**"
                );
    }
}
```

*   修改启动类，开启 AdminServer 及注册发现功能：

```
@EnableDiscoveryClient
@EnableAdminServer
@SpringBootApplication
public class AdminSecurityServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(AdminSecurityServerApplication.class, args);
    }

}
```

*   启动 eureka-server，admin-security-server，访问 Spring Boot Admin 主页发现需要登录才能访问：[http://localhost:9301](http://localhost:9301/)

![](http://www.macrozheng.com/images/springcloud_admin_13.png)

[使用到的模块](#/cloud/admin?id=%e4%bd%bf%e7%94%a8%e5%88%b0%e7%9a%84%e6%a8%a1%e5%9d%97)
---------------------------------------------------------------------------------

```
springcloud-learning
├── eureka-server -- eureka注册中心
├── admin-server -- admin监控中心服务
├── admin-client -- admin监控中心监控的应用服务
└── admin-security-server -- 带登录认证的admin监控中心服务
```

[项目源码地址](#/cloud/admin?id=%e9%a1%b9%e7%9b%ae%e6%ba%90%e7%a0%81%e5%9c%b0%e5%9d%80)
---------------------------------------------------------------------------------

[https://github.com/macrozheng/springcloud-learning](https://github.com/macrozheng/springcloud-learning)

[公众号](#/cloud/admin?id=%e5%85%ac%e4%bc%97%e5%8f%b7)
---------------------------------------------------

![](http://macro-oss.oss-cn-shenzhen.aliyuncs.com/mall/banner/qrcode_for_macrozheng_258.jpg)

* * *