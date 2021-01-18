---
title: 05_Springboot日志
date: 2020-11-17 18:04:00
categories:
- SpringBoot
tags:
- SpringBoot
- 日志
---

<center><font size=4 color="red">05_Springboot日志</font></center>

<!--more-->

## Springboot日志

#### 市面上的日志框架

JUL、JCL、Jboss-logging、logback、log4j、slf4j...

日志分为两部分：**日志门面**和**日志实现**

日志门面：可以理解为就是一个接口，规定了日志所有的功能

日志实现：就是对日志门面的具体实现

| 日志门面（日志的抽象层）                                     | 日志实现                                            |
| ------------------------------------------------------------ | --------------------------------------------------- |
| ~~JCL(Jakarta Commons Logging)~~        SLF4J(Simple Logging Facade for Java)   ~~jboss-logging~~ | Log4j    JUL(Java.util.logging)   Log4j2    Logback |

> 对于框架中如果要使用日志，就必须是：日志门面+日志实现

说一下Springboot使用的日志框架：

Springboot底层是Spring框架，而Spring框架默认使用JCL，但是Springboot并不是使用JCL，它经过一系列对日志的加工处理后，最后使用的框架是：**SLF4j+ Logback**

#### SLF4j的使用

以后开发的时候，日志记录方法的调用，不应该直接调用日志的实现类（日志实现），而应该调用日志抽象层（日志门面）里面的方法

给系统里导入slf4j的jar和logback的实现jar(导入的是jar，但是具体使用导入的包都是日志门面里的包)

```java
package com.hui.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class PersonService {

    Logger logger=LoggerFactory.getLogger(PersonService.class);

    public String personMethod(){
        logger.info("日志记录");
        return "I am a person!";
    }
}
```

日志使用的关系图

![](concrete-bindings.png)

这张图记录了日志的6种使用组合框架

* **SLF4J unbound：**这种只有一个SLF4j日志门面（只引入了slf4j-api.jar），没有日志实现，所有不会有日志打印输出，输出为/dev/null
* **SLF4J bound to logback-calssic：**SLF4j+logback的组合方式，这里引入的jar包有slf4j-api.jar、logback-calssic.jar、logback-core.jar，该日志的输出形式是logback形式
* **SLF4J bound to log4j：**由于log4j出现的比较早，那时候还没有考虑到使用SLF4j日志门面，所以现在如果要使用SLF4j+log4j，需要引入一个中间组件slf4j-log412.jar，该jar包的作用就是为了把SLF4j和log4j连接起来使用，这里使用的jar包就有了slf4j-api.jar、slf4j-log412.jar、log4j.jar，该日志的输出形式是log4j的形式
* **SLF4J bound  to java.util.logging：**SLF4j和java.util.logging的结合使用，和log4j一样，java.util.logging早期也是没有考虑到SLF4j。所以如果要使用SLF4j+java.util.logging，就需要引入中间组件slf4j-jdk14.jar，这里使用的jar包有slf4j-api.jar、slf4j-jdk14.jar、JVM runtime(jre)，日志形式是java.util.logging形式
* **SLF4J bound to simple：**这个是SLF4j日志的简单实现形式，只需要导入slf4j-api.jar和slf4j-simple.jar
* **SLF4J bound to no-operation：**无日志的形式，导入slf4j-api.jar和slf4j-nop.jar，日志输出为/dev/null

> 每一个日志的实现框架都有自己的配置文件。使用slf4j（日志门面）以后，配置文件还是做成**日志实现**框架自己本身的配置文件

#### SLF4J遗留问题解决

Springboot使用的日志框架是SLF4j+logback，但是项目中我们不只使用一个框架，每个框架又都有自己使用的日志框架：Spring（Commons Logging）、Hibernate（Jboss-logging）、Mybatis...

每个日志框架都有一个配置文件，但是这里我想只使用logback的配置文件，因此需要统一日志记录，别的框架我也使用slf4j进行输出。Springboot如何解决呢？参考下图：

![](legacy.png)

这里介绍了3种，我拿第一种**SLF4J bound to logback-classic with redirection of commons-logging,log4j and java.util.logging to SLF4J**为例说明：

我在使用SLF4j+logback日志框架时，我的项目中还使用了其它的框架，而这些框架使用的日志分别是：Commons logging、log4j、java.util.logging，这时候如果我还想使用SLF4j记录日志的话，以log4j为例：

1、排除掉这些jar包，即排除slf4j-log412.jar

2、排除后依赖log4j日志框架的项目框架肯定会报错，这时候我们使用一个中间组件log4j-over-slf4j.jar（对于Commons logging和java util logging，我们排除它本身的日志API后，需要引入jcl-over-slf4j.jar和jul-over-slf4j.jar），该中间件的作用就是将其它日志转为SLF4j

3、再导入SLF4j的其它实现

至此，我们就可以让系统中所有的日志都统一到SLF4j了

> 对于需要排除的jar包，Springboot已经帮我们排除过了

#### Springboot依赖实现日志

Springboot使用spring-boot-starter-logging做日志功能，这里可以看到没有jcl（Commons logging），因为Springboot2.x使用的是Spring5.x以上的版本，Spring5.x已经对jcl改写了

![](logging.png)

#### Springboot日志的使用

先说一下日志级别，常用5个日志级别，且日志级别由低到高分别为：

```
trace<debug<info<warn<error
```

输出日志的内容由多到少的排序

```
trace>debug>info>warn>error
```

* trace： 是追踪，就是程序推进以下，你就可以写个trace输出，所以trace应该会特别多，不过没关系，我们可以设置最低日志级别不让他输出。

* debug： 调试么，我一般就只用这个作为最低级别，trace压根不用。实在没办法就用eclipse或者idea的debug功能就好了么。

* info： 输出一下你感兴趣的或者重要的信息，这个用的最多了。

* warn： 有些信息不是错误信息，但是也要给程序员的一些提示，类似于eclipse中代码的验证不是有error 和warn（不算错误但是也请注意，比如以下depressed的方法）。

* error： 错误信息。用的也比较多。

> Springboot的默认日志级别是info，比默认级别高的日志都会被输出，所以在Springboot里info、warn、error级别的日志都会被输出，如果想看更多的日志，可以定义日志的级别为debug

由于Springboot使用logback作为日志实现，所以配置文件使用的是：logback.xml，但是Springboot也给出了一个日志配置文件logback-spring.xml，另外我们也能直接在application.properties或者application.yml中使用日志的配置信息，这3个文件的优先级高低顺序为：

```
logback.xml>application.properties/yml>logback-spring.xml
```

Springboot默认输出日志到控制台，如果要编写除控制台输出之外的日志文件，则需在application.properties中设置logging.file或logging.path属性

```
注：二者不能同时使用，如若同时使用，则只有logging.file生效
# logging.file如果指定的是文件名，则会在当前项目根目录下生成一个该文件名的日志文件
# logging.file如果指定的是全路径+文件名，则会在指定路径下生成一个日志文件
logging.file=文件名/全路径+文件名
# 会在系统根目录下生成一个日志文件，文件名为默认的spring.log
logging.path=日志文件路径
 
logging.level.包名=指定包下的日志级别
# 日志常用打印规则：%d{yyyy-MM-dd} [%thread] %-5level %logger{50} - %msg%n
logging.pattern.console=日志打印规则
```

#### logback-spring.xml详解

**Spring Boot官方推荐优先使用带有`-spring`的文件名作为你的日志配置（如使用`logback-spring.xml`，而不是`logback.xml`），命名为logback-spring.xml的日志配置文件，将xml放至 src/main/resource下面。**

```
也可以使用自定义的名称，比如logback-config.xml，只需要在application.properties文件中使用logging.config=classpath:logback-config.xml指定即可
```

如果使用了logback-spring.xml，日志框架就不直接加载日志的配置项，有Springboot解析日志配置，可以使用Springboot的高级Profile功能，即：**springProfile标签**，例如：

```xml
<layout class="ch.qos.logback.classic.PatternLayout">
	<springProfile name="dev">
    	<pattern>%d{yyyy-MM-dd} === [%thread] %-5level %logger{50} - %msg%n</pattern>
    </springProfile>
    <springProfile name="!dev">
    	<pattern>%d{yyyy-MM-dd} >>> [%thread] %-5level %logger{50} - %msg%n</pattern>
    </springProfile>
</layout>
```



在讲解 logback-spring.xml之前我们先来了解三个单词：Logger, Appenders and Layouts（记录器、附加器、布局）：**Logback基于三个主要类：Logger，Appender和Layout**。 这三种类型的组件协同工作，使开发人员能够根据消息类型和级别记录消息，并在运行时控制这些消息的格式以及报告的位置。首先给出一个基本的xml配置如下：

```xml
<configuration>
 
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!-- encoders are assigned the type
         ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>
 
  <logger name="chapters.configuration" level="INFO"/>
 
  <!-- Strictly speaking, the level attribute is not necessary since -->
  <!-- the level of the root level is set to DEBUG by default.       -->
  <root level="DEBUG">          
    <appender-ref ref="STDOUT" />
  </root>  
  
</configuration>
```

encoder中最重要就是pattern属性，它负责控制输出日志的格式，这里给出一个示例：

```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) --- [%15.15(%thread)] %cyan(%-40.40(%logger{40})) : %msg%n</pattern>
```

* %d{yyyy-MM-dd HH:mm:ss.SSS}：日期

* %-5level：日志级别

* %highlight()：颜色，info为蓝色，warn为浅红，error为加粗红，debug为黑色

* %thread：打印日志的线程

* %15.15():如果记录的线程字符长度小于15(第一个)则用空格在左侧补齐,如果字符长度大于15(第二个),则从开头开始截断多余的字符 

* %logger：日志输出的类名

* %-40.40()：如果记录的logger字符长度小于40(第一个)则用空格在右侧补齐,如果字符长度大于40(第二个),则从开头开始截断多余的字符

* %cyan：颜色

* %msg：日志输出内容

* %n：换行符

#### 详细的logback-spring.xml示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true">
 
    <!-- appender是configuration的子节点，是负责写日志的组件。 -->
    <!-- ConsoleAppender：把日志输出到控制台 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 默认情况下，每个日志事件都会立即刷新到基础输出流。 这种默认方法更安全，因为如果应用程序在没有正确关闭appender的情况下退出，则日志事件不会丢失。
         但是，为了显着增加日志记录吞吐量，您可能希望将immediateFlush属性设置为false -->
        <!--<immediateFlush>true</immediateFlush>-->
        <encoder>
            <!-- %37():如果字符没有37个字符长度,则左侧用空格补齐 -->
            <!-- %-37():如果字符没有37个字符长度,则右侧用空格补齐 -->
            <!-- %15.15():如果记录的线程字符长度小于15(第一个)则用空格在左侧补齐,如果字符长度大于15(第二个),则从开头开始截断多余的字符 -->
            <!-- %-40.40():如果记录的logger字符长度小于40(第一个)则用空格在右侧补齐,如果字符长度大于40(第二个),则从开头开始截断多余的字符 -->
            <!-- %msg：日志打印详情 -->
            <!-- %n:换行符 -->
            <!-- %highlight():转换说明符以粗体红色显示其级别为ERROR的事件，红色为WARN，BLUE为INFO，以及其他级别的默认颜色。 -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %highlight(%-5level) --- [%15.15(%thread)] %cyan(%-40.40(%logger{40})) : %msg%n</pattern>
            <!-- 控制台也要使用UTF-8，不要使用GBK，否则会中文乱码 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    
	<!-- info 日志-->
    <!-- RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件 -->
    <!-- 以下的大概意思是：1.先按日期存日志，日期变了，将前一天的日志文件名重命名为XXX%日期%索引，新的日志仍然是project_info.log -->
    <!--             2.如果日期没有发生变化，但是当前日志的文件大小超过10MB时，对当前日志进行分割 重命名-->
    <appender name="info_log" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志文件路径和名称-->
        <File>logs/project_info.log</File>
        <!--是否追加到文件末尾,默认为true-->
        <append>true</append>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>DENY</onMatch><!-- 如果命中ERROR就禁止这条日志 -->
            <onMismatch>ACCEPT</onMismatch><!-- 如果没有命中就使用这条规则 -->
        </filter>
        <!--有两个与RollingFileAppender交互的重要子组件。 第一个RollingFileAppender子组件，即RollingPolicy:负责执行翻转所需的操作。
         RollingFileAppender的第二个子组件，即TriggeringPolicy:将确定是否以及何时发生翻转。 因此，RollingPolicy负责什么和TriggeringPolicy负责什么时候.
        作为任何用途，RollingFileAppender必须同时设置RollingPolicy和TriggeringPolicy,但是，如果其RollingPolicy也实现了TriggeringPolicy接口，则只需要显式指定前者。-->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 日志文件的名字会根据fileNamePattern的值，每隔一段时间改变一次 -->
            <!-- 文件名：logs/project_info.2017-12-05.0.log -->
            <!-- 注意：SizeAndTimeBasedRollingPolicy中 ％i和％d令牌都是强制性的，必须存在，要不会报错 -->
            <fileNamePattern>logs/project_info.%d.%i.log</fileNamePattern>
            <!-- 每产生一个日志文件，该日志文件的保存期限为30天, ps:maxHistory的单位是根据fileNamePattern中的翻转策略自动推算出来的,例如上面选用了yyyy-MM-dd,则单位为天
            如果上面选用了yyyy-MM,则单位为月,另外上面的单位默认为yyyy-MM-dd-->
            <maxHistory>30</maxHistory>
            <!-- 每个日志文件到10mb的时候开始切分，最多保留30天，但最大到20GB，哪怕没到30天也要删除多余的日志 -->
            <totalSizeCap>20GB</totalSizeCap>
            <!-- maxFileSize:这是活动文件的大小，默认值是10MB，测试时可改成5KB看效果 -->
            <maxFileSize>10MB</maxFileSize>
        </rollingPolicy>
        <!--编码器-->
        <encoder>
            <!-- pattern节点，用来设置日志的输入格式 ps:日志文件中没有设置颜色,否则颜色部分会有ESC[0:39em等乱码-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level --- [%15.15(%thread)] %-40.40(%logger{40}) : %msg%n</pattern>
            <!-- 记录日志的编码:此处设置字符集 - -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    
    <!-- error 日志-->
    <!-- RollingFileAppender：滚动记录文件，先将日志记录到指定文件，当符合某个条件时，将日志记录到其他文件 -->
    <!-- 以下的大概意思是：1.先按日期存日志，日期变了，将前一天的日志文件名重命名为XXX%日期%索引，新的日志仍然是project_error.log -->
    <!--             2.如果日期没有发生变化，但是当前日志的文件大小超过10MB时，对当前日志进行分割 重命名-->
    <appender name="error_log" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志文件路径和名称-->
        <File>logs/project_error.log</File>
        <!--是否追加到文件末尾,默认为true-->
        <append>true</append>
        <!-- ThresholdFilter过滤低于指定阈值的事件。 对于等于或高于阈值的事件，ThresholdFilter将在调用其decision（）方法时响应NEUTRAL。 但是，将拒绝级别低于阈值的事件 -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level><!-- 低于ERROR级别的日志（debug,info）将被拒绝，等于或者高于ERROR的级别将相应NEUTRAL -->
        </filter>
        <!--有两个与RollingFileAppender交互的重要子组件。 第一个RollingFileAppender子组件，即RollingPolicy:负责执行翻转所需的操作。
        RollingFileAppender的第二个子组件，即TriggeringPolicy:将确定是否以及何时发生翻转。 因此，RollingPolicy负责什么和TriggeringPolicy负责什么时候.
       作为任何用途，RollingFileAppender必须同时设置RollingPolicy和TriggeringPolicy,但是，如果其RollingPolicy也实现了TriggeringPolicy接口，则只需要显式指定前者。-->
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 活动文件的名字会根据fileNamePattern的值，每隔一段时间改变一次 -->
            <!-- 文件名：logs/project_error.2017-12-05.0.log -->
            <!-- 注意：SizeAndTimeBasedRollingPolicy中 ％i和％d令牌都是强制性的，必须存在，要不会报错 -->
            <fileNamePattern>logs/project_error.%d.%i.log</fileNamePattern>
            <!-- 每产生一个日志文件，该日志文件的保存期限为30天, ps:maxHistory的单位是根据fileNamePattern中的翻转策略自动推算出来的,例如上面选用了yyyy-MM-dd,则单位为天
            如果上面选用了yyyy-MM,则单位为月,另外上面的单位默认为yyyy-MM-dd-->
            <maxHistory>30</maxHistory>
            <!-- 每个日志文件到10mb的时候开始切分，最多保留30天，但最大到20GB，哪怕没到30天也要删除多余的日志 -->
            <totalSizeCap>20GB</totalSizeCap>
            <!-- maxFileSize:这是活动文件的大小，默认值是10MB，测试时可改成5KB看效果 -->
            <maxFileSize>10MB</maxFileSize>
        </rollingPolicy>
        <!--编码器-->
        <encoder>
            <!-- pattern节点，用来设置日志的输入格式 ps:日志文件中没有设置颜色,否则颜色部分会有ESC[0:39em等乱码-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level --- [%15.15(%thread)] %-40.40(%logger{40}) : %msg%n</pattern>
            <!-- 记录日志的编码:此处设置字符集 - -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!--给定记录器的每个启用的日志记录请求都将转发到该记录器中的所有appender以及层次结构中较高的appender（不用在意level值）。
    换句话说，appender是从记录器层次结构中附加地继承的。
    例如，如果将控制台appender添加到根记录器，则所有启用的日志记录请求将至少在控制台上打印。
    如果另外将文件追加器添加到记录器（例如L），则对L和L'子项启用的记录请求将打印在文件和控制台上。
    通过将记录器的additivity标志设置为false，可以覆盖此默认行为，以便不再添加appender累积-->
    <!-- configuration中最多允许一个root，别的logger如果没有设置级别则从父级别root继承 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
 
    <!-- 指定项目中某个包，当有日志操作行为时的日志记录级别 -->
    <!-- 级别依次为【从高到低】：FATAL > ERROR > WARN > INFO > DEBUG > TRACE  -->
    <logger name="com.sailing.springbootmybatis" level="INFO">
        <appender-ref ref="info_log" />
        <appender-ref ref="error_log" />
    </logger>
 
    <!-- 利用logback输入mybatis的sql日志，
    注意：如果不加 additivity="false" 则此logger会将输出转发到自身以及祖先的logger中，就会出现日志文件中sql重复打印-->
    <logger name="com.sailing.springbootmybatis.mapper" level="DEBUG" additivity="false">
        <appender-ref ref="info_log" />
        <appender-ref ref="error_log" />
    </logger>
 
    <!-- additivity=false代表禁止默认累计的行为，即com.atomikos中的日志只会记录到日志文件中，不会输出层次级别更高的任何appender-->
    <logger name="com.atomikos" level="INFO" additivity="false">
        <appender-ref ref="info_log" />
        <appender-ref ref="error_log" />
    </logger>
 
</configuration>
```

#### 另一个参考示例

* 自己改下value="G:/logs/pmp"这个值，如果你相关依赖弄好的话，直接复制粘贴即用
* 输出的日志文件的名称最好也改下，下文中<file>${log.path}/web_info.log</file>是因为我这个模块就叫web，要改的话，一个appender改两处
* 集成到springboot的yml格式配置文件的示例：

```xml
logging:
  config: classpath:logback-spring.xml
  level:
    dao: debug
    org:
      mybatis: debug
```

logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文档如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文档是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。
                 当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration  scan="true" scanPeriod="10 seconds">
    <contextName>logback</contextName>

    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义后，可以使“${}”来使用变量。 -->
    <property name="log.path" value="G:/logs/pmp" />

    <!--0. 日志格式和颜色渲染 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
    <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
    <!-- 彩色日志格式 -->
    <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

    <!--1. 输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>debug</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!--2. 输出到文档-->
    <!-- 2.1 level为 DEBUG 日志，时间滚动输出  -->
    <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/web_debug.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志归档 -->
            <fileNamePattern>${log.path}/web-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录debug级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>debug</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.2 level为 INFO 日志，时间滚动输出  -->
    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/web_info.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${log.path}/web-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录info级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>info</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.3 level为 WARN 日志，时间滚动输出  -->
    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/web_warn.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/web-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录warn级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.4 level为 ERROR 日志，时间滚动输出  -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/web_error.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/web-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!--
        <logger>用来设置某一个包或者具体的某一个类的日志打印级别、
        以及指定<appender>。<logger>仅有一个name属性，
        一个可选的level和一个可选的addtivity属性。
        name:用来指定受此logger约束的某一个包或者具体的某一个类。
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
              还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。
              如果未设置此属性，那么当前logger将会继承上级的级别。
        addtivity:是否向上级logger传递打印信息。默认是true。
        <logger name="org.springframework.web" level="info"/>
        <logger name="org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor" level="INFO"/>
    -->

    <!--
        使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
        第一种把<root level="info">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息
        第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：
        【logging.level.org.mybatis=debug logging.level.dao=debug】
     -->

    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        不能设置为INHERITED或者同义词NULL。默认是DEBUG
        可以包含零个或多个元素，标识这个appender将会添加到这个logger。
    -->

    <!-- 4. 最终的策略 -->
    <!-- 4.1 开发环境:打印控制台-->
    <springProfile name="dev">
        <logger name="com.sdcm.pmp" level="debug"/>
    </springProfile>

    <root level="info">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="DEBUG_FILE" />
        <appender-ref ref="INFO_FILE" />
        <appender-ref ref="WARN_FILE" />
        <appender-ref ref="ERROR_FILE" />
    </root>

    <!-- 4.2 生产环境:输出到文档
    <springProfile name="pro">
        <root level="info">
            <appender-ref ref="CONSOLE" />
            <appender-ref ref="DEBUG_FILE" />
            <appender-ref ref="INFO_FILE" />
            <appender-ref ref="ERROR_FILE" />
            <appender-ref ref="WARN_FILE" />
        </root>
    </springProfile> -->

</configuration>
```
