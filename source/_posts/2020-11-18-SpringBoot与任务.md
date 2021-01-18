---
title: 10_SpringBoot与任务
date: 2020-11-18 19:10:28
categories:
- SpringBoot 
tags:
- SpringBoot
- 任务
---

<center><font size=4 color="red">10_SpringBoot与任务</font></center>

<!--more-->

# SpringBoot与任务

## 异步任务

异步任务的处理非常简单，只需要两个注解：

@EnableAsync：配置在启动类上，用于开启异步注解

@Async：配置在需要开启异步的方法上，用于开启异步方法

#### 异步任务示例

**启动类**

```java
//开启异步注解
@EnableAsync
@SpringBootApplication
public class SpringbootesApplication {
}
```

**服务类**

```java
@Service
public class AsyncService {

    //开启异步进程
    @Async
    public void asyncMethod(){
        try {
            Thread.sleep(3000);
            System.out.println("睡3秒后的一个任务开始执行。。。");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**测试类**

com.hui.controller.AsycnController

```java
@RestController
public class AsycnController {

    @Autowired
    private AsyncService asyncService;

    @GetMapping("/hello")
    public String asyncTest(){
        asyncService.asyncMethod();
        return "success";
    }
}
```

启动项目，然后访问：http://localhost:8080/hello

实现结果是：浏览器会立即打印出`success`，然后过了3秒后控制台打印：`睡3秒后的一个任务开始执行。。。`，这便实现了异步的效果

如果我们在这里不适用异步注解，那就是同步，执行的结果是：过3秒后在控制台打印：`睡3秒后的一个任务开始执行。。。`,然后立刻在浏览器上打印出`success`

## 定时任务

定时任务也非常简单，和异步任务一样，只需要两个注解

@EnableScheduling：配置在启动类上，开启定时任务注解

@Scheduled：配置在需要开启定时任务的方法上，用于开启定时任务方法，该注解下常用的一个属性是cron，该属性用于定于定时的时间

#### core表达式

core表达式共有6位，按顺序分别为：秒 分 小时 日期 月份 星期

| 字段 | 允许值                            | 允许的特殊字符                 |
| ---- | --------------------------------- | ------------------------------ |
| 秒   | 0-59                              | ,   -    *   /                 |
| 分   | 0-59                              | ,   -    *   /                 |
| 小时 | 0-23                              | ,   -    *   /                 |
| 日期 | 1-31                              | ,   -    *   ?  /   L   W   C  |
| 月份 | 1-12                              | ,   -    *   /                 |
| 星期 | 0-7或SUN-SAT（其中0和7都表示SUN） | ,   -    *  ?   /   L    C   # |

**特殊字符含义**

| 特殊字符 | 代表含义                   |
| -------- | -------------------------- |
| ,        | 枚举                       |
| -        | 区间                       |
| *        | 任意                       |
| /        | 步长                       |
| ?        | 日/星期冲突匹配            |
| L        | 最后                       |
| W        | 工作日                     |
| C        | 和calendar联系后计算过的值 |
| #        | 星期，4#2表示：第2个星期四 |

举几个关于core的例子

【0 0/5 14,18 * * ?】：每天14点整和18点整，每过5分钟执行一次

【0 15 10 ? * 1-6】：每个月的周一至周六10:15分执行一次

【0 0 2 ? * 6L】: 每个月的最后一个周六凌晨2点执行一次

【0 0 2 LW * ?】：每个月的最后一个工作日凌晨2点执行一次

【0 0 2-4 ? * 1#1】：每个月的第一个周一凌晨2-4点期间，每个整点都执行一次

【0,10,15 * * * * *】：分别在每分钟的第0s、第10s、第15秒执行

【0-2 * * * * *】：每分钟执行3次，分别在0s、1s、2s时执行

【* * * * * *】：每秒钟的任何1s都执行一次

【0/5 * * * * *】：每隔5秒执行一次，执行的具体时刻是0s、5s、10s、15s...

【3/10 * * * * *】：每隔10秒执行一次，执行的具体时刻是：3s、13s、23s...

#### 代码使用

**主配置类**

```java
//开启定时任务注解
@EnableScheduling
@SpringBootApplication
public class SpringbootesApplication {
}
```

**测试类**

com.hui.service.ScheduledService

```java
@Service
public class ScheduledService {

    //每秒钟的第0秒执行一次
    @Scheduled(cron = "0 * * * * *")
    public void scheduledMethod(){
        System.out.println("定时任务执行。。。");
    }
}
```

测试结果：项目启动时：每秒钟的第0秒控制台就会打印一次`定时任务执行。。。`

## 邮件任务

邮件任务依然是非常简单的

引入pom依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

application.properties配置

```properties
# 我这里配置的发送方是网易的163邮箱
# 发送方邮箱名
spring.mail.username=wanghui092901@163.com
# 邮箱密码，不是使用的明码，使用的是授权码，授权码的获取方式见下图
spring.mail.password=RBDKKPTULOUARLFK
# 网易163邮箱的smtp服务器地址
spring.mail.host=smtp.163.com
spring.mail.protocol=smtp
spring.mail.default-encoding=UTF-8
# 如果报ssl的相关错误时，配置为ture就行，我这里是163邮箱发给qq邮箱，并没有报错，就不配了
#spring.mail.properties.mail.smtp.ssl.enable=true
```

获取网易163邮箱的授权码

![](163.jpg)

#### 简单邮箱测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootesApplicationTests {

    @Autowired
    private JavaMailSenderImpl javaMailSender;
    
    @Test
    public void sendMail()  {
        //发送一些简单的邮件内容，包括标题和文本
        SimpleMailMessage mailMessage=new SimpleMailMessage();
        //设置邮件标题
        mailMessage.setSubject("年会通知");
        //设置邮件内容
        mailMessage.setText("地点：XXX大酒店；时间2020-12-12");
        //设置发送给谁，我这里发送给了一个qq邮箱账号
        mailMessage.setTo("1312953501@qq.com");
        //设置谁发送的
        mailMessage.setFrom("wanghui092901@163.com");
        javaMailSender.send(mailMessage);
    }   
}
```

接收到的邮件：

![](simple-qq.jpg)

#### 复杂邮件测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootesApplicationTests {

    @Autowired
    private JavaMailSenderImpl javaMailSender;

    @Test
    public void sendComplexMail() throws MessagingException {
        //创建一个复杂邮件对象
        MimeMessage mimeMessage = javaMailSender.createMimeMessage();
        //创建MimeMessageHelper对象，第二个参数设置为ture表示可以上传文件
        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);

        //设置邮件标题
        helper.setSubject("年会通知");
        //设置邮件内容，并带上html样式，第二个设置设置为ture，表示支持html样式
        helper.setText("亲爱的王子：</br>" +
                "请准时参加年会！！！</br>"+
                "<b style='color:red'>地点：XXX大酒店；时间2020-12-12</b>",true);

        //这里除了发送给131的qq用户外，还发送给自己一份，不这样会被163邮箱认为是垃圾邮件
        String[] strs=new String[]{"1312953501@qq.com","wanghui092901@163.com"};
        //设置发送给谁
        helper.setTo(strs);
        //设置谁发送的
        helper.setFrom("wanghui092901@163.com");

        //设置要发送的文件 1.jpg是发送过去的文件被重新命名为1.jpg了
        helper.addAttachment("1.jpg",new File("D:\\Hui\\Persion\\Docs\\MarkDown\\Springboot\\media\\163.jpg"));
        helper.addAttachment("2.jpg",new File("D:\\Hui\\Persion\\Docs\\MarkDown\\Springboot\\media\\simple-qq.jpg"));

        javaMailSender.send(mimeMessage);

    }
}
```

> 发邮件时注意邮件格式和内容，否则可能会被认为是垃圾邮件！！！

接收到的邮件：

![](complex-qq.jpg)

#### 使用邮件模板

邮件内容在代码中写html，肯定是不合理的，因此我们可以使用邮件模板来定义邮件内容

引入邮件模板依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

在resources资源路径的templates下创建名mail.ftl的模板

```html
<html>
    <body>
        <h3>${username}你好：</h3>
        <p>请准时参加今晚的年会!!!</p>
        <span>地点：</span><b style='color:green'>${address}</b>
        <span>时间：</span><b style='color:green'>${time}</b>
    </body>
</html>
```

编写邮件测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootesApplicationTests {

    @Autowired
    private JavaMailSenderImpl javaMailSender;

    @Test
    public void sendTemplateMail() throws Exception{
        MimeMessage mimeMessage = javaMailSender.createMimeMessage();

        MimeMessageHelper helper = new MimeMessageHelper(mimeMessage, true);

        //这里除了发送给131的qq用户外，还发送给自己一份，不这样会被163邮箱认为是垃圾邮件
        String[] strs=new String[]{"1312953501@qq.com","wanghui092901@163.com"};
        //设置发送给谁
        helper.setTo(strs);
        //设置谁发送的
        helper.setFrom("wanghui092901@163.com");

        Configuration configuration = new Configuration(Configuration.VERSION_2_3_28);
        //指定模板所在路径
        configuration.setClassForTemplateLoading(this.getClass(), "/templates");

        //将模板中的参数映射到params中
        Map<String,String> params=new HashMap<>();
        params.put("username","王子");
        params.put("address","乌鲁木齐大酒店");
        params.put("time","晚上10点");
        //使用模板并给模板里的参数赋值
        String mailText = FreeMarkerTemplateUtils.processTemplateIntoString(configuration.getTemplate("mail.ftl"), params);

        //设置邮件标题
        helper.setSubject("年会通知");
        //设置邮件内容，并带上html样式，第二个设置设置为ture，表示支持html样式
        helper.setText(mailText, true);

        javaMailSender.send(mimeMessage);
    }
}
```

接收到的邮件：

![](template-qq.jpg)


