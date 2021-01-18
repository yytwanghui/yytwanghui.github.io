---
title: 03_Springboot启动配置原理
date: 2020-11-17 17:58:10
categories:
- SpringBoot
tags:
- SpringBoot
- 启动配置原理
---

<center><font size=4 color="red">03_Springboot启动配置原理</font></center>

<!--more-->

# Springboot启动配置原理

## 启动配置原理

#### 核心启动配置参数

```
配置在META-INF/spring.factories中
ApplicationContextInitializer
SpringApplicationRunListener

需要放在IOC容器中
ApplicationRunner
CommandLineRunner
```

#### 主要步骤

* 准备环境
  * 执行ApplicationContextInitializer.initialize()
  * 监听器SpringApplicationRunListener回调contextPrepared
  * 加载主配置类定义信息
  * 监听器SpringApplicationRunListener回调contextLoaded
* 刷新启动IOC容器
  * 扫描加载所有容器中的组件
  * 包括从MEAT-INF/spring.factories中获取所有的EnableAutoConfiguration组件
* 回调容器中所有的ApplicationRunner和CommandLineRunner的run方法
* 监听器SpringApplicationRunListener回调finished

## 具体流程步骤

首先对启动流程打上断点然后以Dubug模式运行

![](run.jpg)

点击`Step Into`向下运行，可以看到以下一个方法

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return (new SpringApplication(primarySources)).run(args);
}
```

这个方法主要有两部分，第一部分创建SpringApplication对象，第二部分执行run方法

#### 创建SpringApplication对象

`Step Over`到`new SpringApplication(primarySources))`可以看到创建SpringApplication对象的过程

```java
public SpringApplication(Class<?>... primarySources) {
    this((ResourceLoader)null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 保存主配置类
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    // 判断当前是否一个web应用
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
     // 从类路径下找到META-INF/spring.factories配置的所有ApplicationContextInitializer；然后保存起来。
  this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 从类路径下找到META-INF/spring.factories配置的所有ApplicationListener
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    //从多个配置类中找到有main方法的主配置类,就是Springboot的启动类
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

#### 执行run方法

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
    this.configureHeadlessProperty();
    // 获取SpringApplicationRunListener; 从类路径下META-INF/spring.factories获取
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    // 回调所有的获取SpringApplicationRunListener.starting()方法
    listeners.starting();

    Collection exceptionReporters;
    try {
        // 封装命令行参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备环境,创建环境完成后回调SpringApplicationRunListener.environmentPrepared();表示环境准备完成
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
        this.configureIgnoreBeanInfo(environment);
        //打印Springboot的启动标志logo
        Banner printedBanner = this.printBanner(environment);
        // 创建ApplicationContext；决定创建web的IOC还是普通的IOC
        context = this.createApplicationContext();
        exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
        // 准备上下文环境；将environment保存到IOC中，而且执行applyInitializers();
        // applyInitializers():回调之前保存的所有ApplicationContextInitializer的initialize方法
        // 回调所有的SpringApplicationRunListener的contextPrepared();
        this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // prepareContext运行完成以后回调所有的SpringApplicationRunListener的contextLoaded();
        // 扫描容器；IOC容器初始化（如果是web应用还会创建嵌入的Tomcat）；
        // 扫描，创建，加载所有组件的地方(配置类，组件，自动配置)
        this.refreshContext(context);
        // 从IOC容器中获取所有的ApplicationRunner和CommandLineRunner进行回调
        // ApplicationRunner先回调，CommandLineRunner再回调
        this.afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }

        listeners.started(context);
        this.callRunners(context, applicationArguments);
    } catch (Throwable var10) {
        this.handleRunFailure(context, var10, exceptionReporters, listeners);
        throw new IllegalStateException(var10);
    }

    try {
        listeners.running(context);
        // 整个SpringBoot应用启动完成以后返回启动的IOC容器
        return context;
    } catch (Throwable var9) {
        this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var9);
    }
}
```

## 事件监听机制

模拟事件监听机制，主要是以下几个类：

```
配置在META-INF/spring.factories中
ApplicationContextInitializer
SpringApplicationRunListener

需要放在IOC容器中
ApplicationRunner
CommandLineRunner
```

#### ApplicationContextInitializer

配置在META-INF/spring.factories中

```java
package com.hui.listeners;

import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;

public class CustomApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
        System.out.println("ApplicationContextInitializer.....initialize");
    }
}
```

#### SpringApplicationRunListener

配置在META-INF/spring.factories中

```java
package com.hui.listeners;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringApplicationRunListener;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.core.env.ConfigurableEnvironment;

public class CustomSpringApplicationRunListener implements SpringApplicationRunListener {

    // 必须有的构造器
    public CustomSpringApplicationRunListener(SpringApplication application, String[] args){

    }

    @Override
    public void starting() {
        System.out.println("SpringApplicationRunListener...starting....");
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        Map<String, Object> systemEnvironment = environment.getSystemEnvironment();

        for (Map.Entry<String, Object> key_value : systemEnvironment.entrySet()) {
            System.out.println(key_value.getKey()+"==="+key_value.getValue());
        }
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println("SpringApplicationRunListener..contextPrepared");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println("SpringApplicationRunListener..contextLoaded");
    }

    @Override
    public void started(ConfigurableApplicationContext context) {
        System.out.println("SpringApplicationRunListener..started");
    }

    @Override
    public void running(ConfigurableApplicationContext context) {
        System.out.println("SpringApplicationRunListener..running");
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println("SpringApplicationRunListener..failed");
    }
}
```

#### 添加META-INF/spring.factories配置

在resources下创建包META-INF，在包下创建文件spring.factories，然后将CustomApplicationContextInitializer和CustomSrpingApplicationRunListener配置进去

```factories
# Initializers
org.springframework.context.ApplicationContextInitializer=\
com.hui.listeners.CustomApplicationContextInitializer

# SpringApplicationRunListener Listeners
org.springframework.boot.SpringApplicationRunListener=\
com.hui.listeners.CustomSpringApplicationRunListener
```

#### ApplicationRunner

需要放在IOC容器中

```java
package com.hui.listeners;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component
public class CustomApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("applicationRunner ... run ...");
    }
}
```

#### CommandLineRunner

需要放在IOC容器中

```java
package com.hui.listeners;

import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class CustomCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("commandLineRunner ... run ...");
    }
}
```

#### 启动项目

启动项目，在控制台可以看到监听器的执行顺序

```powershell
SpringApplicationRunListener...starting....
ACTEL_FOR_ALTIUM_OVERRIDE=== 
USERDOMAIN_ROAMINGPROFILE===DESKTOP-COC7777
PROCESSOR_LEVEL===6
SESSIONNAME===Console
ALLUSERSPROFILE===C:\ProgramData
PROCESSOR_ARCHITECTURE===AMD64
GIT_INSTALL_ROOT===C:\Users\wanghui\scoop\apps\git\current
PSModulePath===C:\Program Files\WindowsPowerShell\Modules;C:\Windows\system32\WindowsPowerShell\v1.0\Modules
SystemDrive===C:
MOZ_PLUGIN_PATH===C:\Users\wanghui\scoop\apps\foxit-reader\current\plugins\
USERNAME===wanghui
ProgramFiles(x86)===C:\Program Files (x86)
FPS_BROWSER_USER_PROFILE_STRING===Default
PATHEXT===.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
Cmder===C:\Tools\cmder
DriverData===C:\Windows\System32\Drivers\DriverData
ProgramData===C:\ProgramData
ProgramW6432===C:\Program Files
HOMEPATH===\Users\wanghui
PROCESSOR_IDENTIFIER===Intel64 Family 6 Model 158 Stepping 10, GenuineIntel
ProgramFiles===C:\Program Files
PUBLIC===C:\Users\Public
windir===C:\Windows
=::===::\
OneDriveCommercial===C:\Users\wanghui\OneDrive - students.solano.edu
LOCALAPPDATA===C:\Users\wanghui\AppData\Local
IntelliJ IDEA===C:\tools\JetBrains\IntelliJ IDEA 2019.3.3\bin;
USERDOMAIN===DESKTOP-COC7777
FPS_BROWSER_APP_PROFILE_STRING===Internet Explorer
LOGONSERVER===\\DESKTOP-COC7777
JAVA_HOME===C:\Users\wanghui\scoop\apps\ojdkbuild8-full\current
ALTERA_FOR_ALTIUM_OVERRIDE=== 
CATALINA_BASE===C:\Users\wanghui\scoop\apps\tomcat8\current
OneDrive===C:\Users\wanghui\OneDrive - students.solano.edu
APPDATA===C:\Users\wanghui\AppData\Roaming
CommonProgramFiles===C:\Program Files\Common Files
Path===C:\Tools\Xshell\;C:\Tools\Xmanager\;C:\Program Files (x86)\Intel\Intel(R) Management Engine Components\iCLS\;C:\Program Files\Intel\Intel(R) Management Engine Components\iCLS\;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files (x86)\Intel\Intel(R) Management Engine Components\DAL;C:\Program Files\Intel\Intel(R) Management Engine Components\DAL;C:\Tools\Git\cmd;C:\Tools\apache-maven-3.6.3\bin;C:\Tools\java18\jdk1.8.0_161\bin;C:\Users\wanghui\scoop\apps\nodejs12\current\bin;C:\Users\wanghui\scoop\apps\nodejs12\current;C:\Users\wanghui\scoop\apps\maven\current\bin;C:\Users\wanghui\scoop\apps\ojdkbuild8-full\current\bin;C:\Users\wanghui\scoop\shims;C:\Users\wanghui\AppData\Local\Microsoft\WindowsApps;C:\Users\wanghui\AppData\Local\Pandoc\;C:\Tools\bandzip\;C:\tools\JetBrains\IntelliJ IDEA 2019.3.3\bin;
OS===Windows_NT
COMPUTERNAME===DESKTOP-COC7777
CATALINA_HOME===C:\Users\wanghui\scoop\apps\tomcat8\current
PROCESSOR_REVISION===9e0a
CommonProgramW6432===C:\Program Files\Common Files
ComSpec===C:\Windows\system32\cmd.exe
SystemRoot===C:\Windows
TEMP===C:\Users\wanghui\AppData\Local\Temp
HOMEDRIVE===C:
USERPROFILE===C:\Users\wanghui
TMP===C:\Users\wanghui\AppData\Local\Temp
CommonProgramFiles(x86)===C:\Program Files (x86)\Common Files
NUMBER_OF_PROCESSORS===6
IDEA_INITIAL_DIRECTORY===C:\Users\wanghui\Desktop

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::       (v2.1.13.RELEASE)

ApplicationContextInitializer.....initialize
SpringApplicationRunListener..contextPrepared
2020-08-24 02:56:36.255  INFO 4544 --- [           main] com.hui.SpringbootdemoApplication        : Starting SpringbootdemoApplication on DESKTOP-COC7777 with PID 4544 (started by wanghui in D:\Hui\Persion\Application\IntelliJ IDEA\springbootdemo)
2020-08-24 02:56:36.257  INFO 4544 --- [           main] com.hui.SpringbootdemoApplication        : No active profile set, falling back to default profiles: default
SpringApplicationRunListener..contextLoaded
2020-08-24 02:56:36.827  INFO 4544 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-08-24 02:56:36.840  INFO 4544 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-08-24 02:56:36.841  INFO 4544 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.31]
2020-08-24 02:56:36.897  INFO 4544 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-08-24 02:56:36.897  INFO 4544 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 616 ms
2020-08-24 02:56:37.006  INFO 4544 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-08-24 02:56:37.107  INFO 4544 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-08-24 02:56:37.109  INFO 4544 --- [           main] com.hui.SpringbootdemoApplication        : Started SpringbootdemoApplication in 1.085 seconds (JVM running for 1.567)
SpringApplicationRunListener..started
applicationRunner ... run ...
commandLineRunner ... run ...
SpringApplicationRunListener..running

```

## Springboot自定义starter

#### 如何编写自动配置 

我们参照@WebMvcAutoConfiguration为例，我们看看们需要准备哪些东西，下面是WebMvcAutoConfiguration的部分代码：

```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurerAdapter.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {

    @Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})
    @EnableConfigurationProperties({WebMvcProperties.class, ResourceProperties.class})
    public static class WebMvcAutoConfigurationAdapter extends WebMvcConfigurerAdapter {

        @Bean
        @ConditionalOnBean({View.class})
        @ConditionalOnMissingBean
        public BeanNameViewResolver beanNameViewResolver() {
            BeanNameViewResolver resolver = new BeanNameViewResolver();
            resolver.setOrder(2147483637);
            return resolver;
        }
    }
}
```

我们可以抽取到我们自定义starter时同样需要的一些配置。

```java
@Configuration  //指定这个类是一个配置类
@ConditionalOnXXX  //指定条件成立的情况下自动配置类生效
@AutoConfigureOrder  //指定自动配置类的顺序
@Bean  //向容器中添加组件
@ConfigurationProperties  //结合相关xxxProperties来绑定相关的配置
@EnableConfigurationProperties  //让xxxProperties生效加入到容器中

自动配置类要能加载需要将自动配置类，配置在META-INF/spring.factories中
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
```

#### 模式

我们参照 **spring-boot-starter-web** 我们发现其中没有代码，但是其下的依赖里是有代码的：

![](starter模式.jpg)

我们做自己的自动配置也采用这种模式，就是我们调用的**XXX-spring-boot-starter**启动器里不写代码，只是一个父依赖，我们要写的自动配置类**XXX-spring-boot-autoconfigure**里才写我们需要的代码

#### 自定义starter实例

我们需要实现的工程：在主配置文件中配置一些参数，这些参数能生效

我们需要先创建两个工程 **hello-spring-boot-starter** 和 **hello-spring-boot-starter-autoconfigurer**

**hello-spring-boot-starter**：是maven工程，只是一份父依赖工程，pom文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>hello-spring-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>com.hui</groupId>
            <artifactId>hello-spring-boot-starter-autoconfigurer</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

**hello-spring-boot-starter-autoconfigurer**：是一个Springboot工程，pom文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.13.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.hui</groupId>
    <artifactId>hello-spring-boot-starter-autoconfigurer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>hello-spring-boot-starter-autoconfigurer</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <!--所有的starter都需要引入spring-boot-starter-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

    </dependencies>

</project>
```

删除：启动类、resources下的文件，test文件

在**hello-spring-boot-starter-autoconfigurer**下写需要的服务和自动配置文件

**HelloProperties**

```java
package com.hui.starter;

import org.springframework.boot.context.properties.ConfigurationProperties;

//将配置文件中所有wh.hello的配置映射到HelloProperties配置类中
@ConfigurationProperties(prefix = "wh.hello")
public class HelloProperties {

    private String before;
    private String after;

    public String getBefore() {
        return before;
    }

    public void setBefore(String before) {
        this.before = before;
    }

    public String getAfter() {
        return after;
    }

    public void setAfter(String after) {
        this.after = after;
    }
}
```

**HelloService**

```java
package com.hui.starter;

public class HelloService {

    HelloProperties helloProperties;

    public HelloProperties getHelloProperties() {
        return helloProperties;
    }

    public void setHelloProperties(HelloProperties helloProperties) {
        this.helloProperties = helloProperties;
    }

    public String sayHello(String name ) {
        return helloProperties.getBefore()+ "-" + name + "-" +helloProperties.getAfter();
    }
}
```

**HelloServiceAutoConfiguration**

```java
package com.hui.starter;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnWebApplication //web应用时生效，其实就是工程依赖中引入spring-boot-starter-web
@EnableConfigurationProperties(HelloProperties.class)
public class HelloServiceAutoConfiguration {

    @Autowired
    HelloProperties helloProperties;

    @Bean
    public HelloService helloService() {
        HelloService service = new HelloService();
        service.setHelloProperties(helloProperties);
        return service;
    }

}
```

**spring.factories**

在 **resources** 下创建文件夹 **META-INF** 并在 **META-INF** 下创建文件 **spring.factories** ，内容如下：

```bash
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.gf.service.HelloServiceAutoConfiguration
```

到这儿，我们的配置自定义的starter就写完了 ，我们将hello-spring-boot-starter-autoconfigurer、hello-spring-boot-starter 先后安装成本地jar包。

#### 测试自定义starter

随便写一个Springboot的测试项目

**pom文件**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.13.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.hui</groupId>
    <artifactId>springbootactivemq</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springbootdemo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <targetJavaProject>src/main/java</targetJavaProject>
        <!-- XML生成路径 -->
        <targetResourcesProject>src/main/resources</targetResourcesProject>
        <targetXMLPackage>mapper</targetXMLPackage>
        <targetMapperPackage>com.hui.dao</targetMapperPackage>
        <targetModelPackage>com.hui.pojo</targetModelPackage>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--引入我们自定义的starter-->
        <dependency>
            <groupId>org.hui</groupId>
            <artifactId>hello-spring-boot-starter</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

</project>
```

**HelloController**

```java
package com.hui.controller;

import com.hui.starter.HelloService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @Autowired
    private HelloService helloService;

    @GetMapping("/hello/{name}")
    public String hello(@PathVariable(value = "name") String name) {
        return helloService.sayHello(name);
    }

}
```

**application.yml**

```yaml
wh:
  hello:
    before: wang
    after: hui
```

启动项目，运行`http://localhost:8080/hello/xiao`访问，我的运行结果：

```
wang-xiao-hui
```
