---
title: 06_Springboot数据访问
date: 2020-11-17 18:07:17
categories:
- SpringBoot
tags:
- SpringBoot
- 数据访问
---

<center><font size=4 color="red">06_Springboot数据访问</font></center>

<!--more-->

# Springboot数据访问

## Springboot配置Druid

**pom.xml配置**

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.41</version>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.0.9</version>
</dependency>

<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

**数据库加密**

```powershell
java –cp druid-1.0.18.jar com.alibaba.druid.filter.config.ConfigTools 你的密码
```

![](druid-password.jpg)

**yml.properties配置**

```yaml
server:
  port: 8080

#数据库密码加密
password: Biyu5YzU+6sxDRbmWEa3B2uUcImzDo0BuXjTlL505+/pTb+/0Oqd3ou1R6J8+9Fy3CYrM18nBDqf6wAaPgUGOg==

spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: ${password}
    url: jdbc:mysql://192.168.31.140:3306/test?useUnicode=true&amp;characterEncoding=utf-8&autoReconnect=true&useSSL=false
    type: com.alibaba.druid.pool.DruidDataSource

    # 初始化大小，最小，最大
    initialSize: 5
    minIdle: 5
    maxActive: 20
    # 配置获取连接等待超时的时间
    maxwait: 60000
    # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
    time-between-eviction-runs-millis: 60000
    # 配置一个连接在池中最小生存的时间，单位是毫秒
    min-evictable-idle-time-millis: 300000
    validation-query: SELECT 1 FROM DUAL
    test-while-idle: true
    test-on-borrow: false
    test-on-return: false
    pool-prepared-statements: true
    # 配置监控拦截系统，去掉后监控界面sql无法统计，'wall'用于防火墙
    filters: stat,wall,log4j,config
    maxPoolPreparedStatementPerConnectionSize: 20
    useGlobalDataSourceStat: true
    # 通过connectProperties属性来打开mergeSql功能；慢SQL记录
    connection-properties: durid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
    
mybatis:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    # 开启驼峰转换
    map-underscore-to-camel-case: true
```

**config配置类**

由于spring boot 目前还不直接支持druid，所以需要手动配置DataSource

src/main/java/com/hui/config/DruidConfig.java

```java
package com.hui.config;

import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.support.http.StatViewServlet;
import com.alibaba.druid.support.http.WebStatFilter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class DruidConfig {

    /**
     * 由于spring boot 目前还不直接支持druid，所以需要手动配置DataSource
     * @return
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druid(){
        return new DruidDataSource();
    }


    /**
     * 配置Druid的监控，配置一个管理后台的servlet
     * @return
     */
    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean bean=new ServletRegistrationBean(new StatViewServlet(),"/druid/*");
        Map<String,String> initParams=new HashMap<>();

        //所有需要配置的参数来源于类：ResourceServlet
        initParams.put("loginUsername","admin"); //登录druid监控后台的登录名
        initParams.put("loginPassword","123456"); //登录druid监控后台的密码

        //允许哪个ip可以登录，不写默认是允许所有的ip主机均可访问
        initParams.put("allow","");
        //禁止哪个ip主机登录，以下配置表示192.168.15.21的主机不能登录
        initParams.put("deny","192.168.15.21");

        bean.setInitParameters(initParams);
        return bean;
    }

    /**
     * 配置Druid的监控，配置一个web监控的filter
     * @return
     */
    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean=new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());

        Map<String,String> initParams=new HashMap<>();
        //exclusions表示哪些请求不被拦截
        initParams.put("exclusions","*.js,*.css,/druid/*");

        bean.setInitParameters(initParams);

        //拦截所有的请求
        bean.setUrlPatterns(Arrays.asList("/*"));

        return bean;
    }
}
```

**Druid Web地址**

启动Springboot项目，然后访问：http://localhost:8080/druid/index.html

用户名：admin      密码：123456

![](druidweb.jpg)

## Mybatis逆向工程+JPA

#### 只操作JPA

逆向工程生成的User类需要做修改，加上一下注解

```java
@Entity
@Table(name = "user")
public class User {

    @Id    //主键
    @GeneratedValue(strategy = GenerationType.IDENTITY)   //设置主键生成策略为自增
    private Integer userid;

    @Column(name = "username")    //列名
    private String username;

    @Column(name = "userage")
    private Integer userage;

    @Column(name = "date")
    private Date date;
```

编写UserRepository接口继承JpaRepository

JpaRepository中的参数：

User：对应的pojo类

Integer：对应的主键

```java
package com.hui.dao;

import com.hui.pojo.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Integer> {
}
```

直接使用JpaRepository中的方法

```java
@RestController
public class UserController{

    @Autowired
    private UserRepository userRepository;

    @GetMapping("/userone/{userId}")
    public User findUserOne(@PathVariable Integer userId){
        return userRepository.findById(userId).orElse(null);
    }
}
```

注意：这里不能使用方法：userRepository.findOne(userId)；会报错，原因应该Springboot版本问题，低版本可以用，但是2.0以后的版本不行

#### Mybatis+JPA的整合

暂时没有整合成功，这个后续再研究
