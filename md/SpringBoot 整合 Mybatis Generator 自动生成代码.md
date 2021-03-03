> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/WRb6ayIkG9ennLbJgIvRXQ)

SpringBoot 整合 Mybatis Generator 自动生成代码
======================================

Mybatis 是目前主流的 ORM 框架，相比于 hibernate 的全自动，它是半自动化需要手写 sql 语句、接口、实体对象，后来推出的 Generator 自动生成代码, 可以帮我们提高开发效率。  

> 本文目的：SpringBoot 整合 Mybatis Generator 自动生成 dao、entity、mapper.xml 实现单表增删改查。

##### 1. 创建 SpringBoot 项目

File→New→Project… 选择 Spring Initializr，选择 JDK 版本，默认初始化 URL

![](https://mmbiz.qpic.cn/mmbiz_png/CfiaDgvyLx05X9r6ukKyVNibf92iaC6zWic4Fuu8b4xdxcrNGluIFGDA3ia0UCUHT2CWEkib0kPmFGvABZNm5sOF36hw/640?wx_fmt=png)

填写项目名称，java 版本，其他描述信息

![](https://mmbiz.qpic.cn/mmbiz_png/CfiaDgvyLx05X9r6ukKyVNibf92iaC6zWic4DHaER23QEFfrT8sPWPdXlGUxBYegpSiacmObGNg2SMDLPGhwBpAicYoA/640?wx_fmt=png)

选择 web、mybatis、mysql 依赖  

![](https://mmbiz.qpic.cn/mmbiz_png/CfiaDgvyLx05X9r6ukKyVNibf92iaC6zWic48fibibmQlG2YOoYbVoWm1zZHiah4Ogkm0BcgbficKNMrT8mozaICBsRoUQ/640?wx_fmt=png)

选择项目存放路径  

![](https://mmbiz.qpic.cn/mmbiz_png/CfiaDgvyLx05X9r6ukKyVNibf92iaC6zWic4p8YhChF85icSK6RU8GT9rb7w81XAVDWtF1iaqpWLtY7jkMAFM5EE0GmA/640?wx_fmt=png)

**Finish** 完成项目创建  

##### 2. mybatis-generator-maven 插件的配置

打开项目的 pom.xml 文件添加

```
<plugin>
<groupId>org.mybatis.generator</groupId>
<artifactId>mybatis-generator-maven-plugin</artifactId>
<configuration>
<verbose>true</verbose>
<overwrite>true</overwrite>
</configuration>
</plugin>
```

完整 pom.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>

<groupId>com.xyz</groupId>
<artifactId>mybatis</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>

<name>mybatis</name>
<description>Spring Boot 整合 Mybatis Generator自动生成代码</description>

<parent>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-parent</artifactId>
<version>2.0.5.RELEASE</version>
<relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
<java.version>1.8</java.version>
</properties>

<dependencies>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
<groupId>org.mybatis.spring.boot</groupId>
<artifactId>mybatis-spring-boot-starter</artifactId>
<version>1.3.2</version>
</dependency>

<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<scope>runtime</scope>
</dependency>
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-test</artifactId>
<scope>test</scope>
</dependency>
<dependency>
<groupId>com.github.pagehelper</groupId>
<artifactId>pagehelper-spring-boot-starter</artifactId>
<version>1.2.5</version>
</dependency>
</dependencies>

<build>
<plugins>
<plugin>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
<plugin>
<groupId>org.mybatis.generator</groupId>
<artifactId>mybatis-generator-maven-plugin</artifactId>
<configuration>
<verbose>true</verbose>
<overwrite>true</overwrite>
</configuration>
</plugin>
</plugins>
</build>
</project>
```

##### 3. 项目结构构建

在项目目录下 (这里是 mybatis) 添加 controller、service、dao、entity 包，在 resources 下添加 mapper 包存放映射文件。![](https://mmbiz.qpic.cn/mmbiz_png/CfiaDgvyLx05X9r6ukKyVNibf92iaC6zWic4tZ3hia4ic5ibCKNUMWUc63iaoLZuadm0iaPtD81gEu0iaiaJDAKib6jcxgvt5g/640?wx_fmt=png)

##### 4. application.yml 配置

```
#端口号配置
server:
port: 8080
spring:
#模板引擎配置
thymeleaf:
  prefix: classpath:/templates/
  suffix: .html
  mode: HTML
  encoding: UTF-8
  cache: false
  servlet:
    content-type: text/html
#静态文件配置
resources:
  static-locations: classpath:/static,classpath:/META-INF/resources,classpath:/templates/
#jdbc配置
datasource:
  url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf8
  username: xyz
  password: xyz
  driver-class-name: com.mysql.jdbc.Driver
#mybatis配置
mybatis:
#映射文件路径
mapper-locations: classpath:mapper/*.xml
#模型所在的保命
type-aliases-package: com.magic.mybatis.entity
```

##### 5. generatorConfig.xml 配置

在 resources 文件下创建 generatorConfig.xml 文件，配置如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
       "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<!-- 配置生成器 -->
<generatorConfiguration>

   <!--classPathEntry:数据库的JDBC驱动,换成你自己的驱动位置 可选 -->
   <classPathEntry location="D:\tools\maven\repository\mysql-connector-java-5.1.47.jar"/>

   <!-- 一个数据库一个context,defaultModelType="flat" 大数据字段，不分表 -->
   <context id="MysqlTables" targetRuntime="MyBatis3Simple" defaultModelType="flat">

       <!-- 自动识别数据库关键字，默认false，如果设置为true，根据SqlReservedWords中定义的关键字列表；一般保留默认值，遇到数据库关键字（Java关键字），使用columnOverride覆盖 -->
       <property />

       <!-- 生成的Java文件的编码 -->
       <property />

       <!-- beginningDelimiter和endingDelimiter：指明数据库的用于标记数据库对象名的符号，比如ORACLE就是双引号，MYSQL默认是`反引号；-->
       <property `"/>
       <property `"/>

       <!-- 格式化java代码 -->
       <property />

       <!-- 格式化XML代码 -->
       <property />
       <plugin type="org.mybatis.generator.plugins.SerializablePlugin"/>
       <plugin type="org.mybatis.generator.plugins.ToStringPlugin"/>

       <!-- 注释 -->
       <commentGenerator>
           <property /><!-- 是否取消注释 -->
           <property /> <!-- 是否生成注释代时间戳-->
       </commentGenerator>

       <!-- jdbc连接-->
       <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                       connectionURL="jdbc:mysql://localhost:3306/test?serverTimezone=UTC" userId="magic"
                       password="****"/>

       <!-- 类型转换 -->
       <javaTypeResolver>
           <!-- 是否使用bigDecimal， false可自动转化以下类型（Long, Integer, Short, etc.） -->
           <property />
       </javaTypeResolver>

       <!-- 生成实体类地址 -->
       <javaModelGenerator targetPackage="com.magic.mybatis.entity" targetProject="src/main/java">
           <!-- 是否让schema作为包的后缀 -->
           <property />
           <!-- 从数据库返回的值去掉前后空格 -->
           <property />
       </javaModelGenerator>

       <!-- 生成map.xml文件存放地址 -->
       <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources">
           <property />
       </sqlMapGenerator>

       <!-- 生成接口dao -->
       <javaClientGenerator targetPackage="com.magic.mybatis.dao" targetProject="src/main/java" type="XMLMAPPER">
           <property />
       </javaClientGenerator>

       <!-- table可以有多个,每个数据库中的表都可以写一个table，tableName表示要匹配的数据库表,也可以在tableName属性中通过使用%通配符来匹配所有数据库表,只有匹配的表才会自动生成文件 enableSelectByPrimaryKey相应的配置表示是否生成相应的接口 -->
       <table table
              enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false"
              enableSelectByPrimaryKey="true" enableUpdateByPrimaryKey="true"
              enableDeleteByPrimaryKey="true">
           <property />
       </table>

   </context>
</generatorConfiguration>
```

> 注意：classPathEntry location=“D:\tools\maven\repository\mysql-connector-java-5.1.47.jar”，建议用 5.X 系列的，否则可能生成的接口会缺少

##### 6. 在 idea 右侧选择 maven > 项目名 > Plugins > Mybatis-generator > Mybatis-Generator:generate。

##### 7. 点击 Mybatis-Generator:generate，自动在 dao、entity、mapper 包下生成代码

> 注意：利用 Mybatis Generator 自动生成代码，对于已经存在的文件会存在覆盖和在原有文件上追加的可能性，不宜多次生成。如需重新生成，需要删除已生成的源文件