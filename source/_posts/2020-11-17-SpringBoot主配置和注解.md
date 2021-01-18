---
title: 04_SpringBoot主配置和注解
date: 2020-11-17 18:01:25
categories:
- SpringBoot
tags:
- SpringBoot
- 主配置
- 注解
---

<center><font size=4 color="red">04_SpringBoot主配置和注解</font></center>

<!--more-->

# SpringBoot主配置和注解

建议Springboot使用版本：<version>2.1.13.RELEASE</version>

applicatin.properties的优先级比application.yml优先级高，但是一般项目中不会同时出现这两种配置文件

## yml配置文件写法

例子：

person是对象，maps是map集合，lists是list集合，student是对象，具体关系为：

```java
package com.hui.pojo;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.List;
import java.util.Map;

@ConfigurationProperties(prefix = "person")
@Component
public class Person {

    private String name;
    private String age;
    private Date birth;
    private Map<String,Object> maps;
    private List<String> lists;
    private Student student;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    public Date getBirth() {
        return birth;
    }

    public void setBirth(Date birth) {
        this.birth = birth;
    }

    public Map<String, Object> getMaps() {
        return maps;
    }

    public void setMaps(Map<String, Object> maps) {
        this.maps = maps;
    }

    public List<String> getLists() {
        return lists;
    }

    public void setLists(List<String> lists) {
        this.lists = lists;
    }

    public Student getStudent() {
        return student;
    }

    public void setStudent(Student student) {
        this.student = student;
    }

    @Override
    public String toString() {
        return "Persion{" +
                "name='" + name + '\'' +
                ", age='" + age + '\'' +
                ", birth=" + birth +
                ", maps=" + maps +
                ", lists=" + lists +
                ", student=" + student +
                '}';
    }
}
```

```java
package com.hui.pojo;

public class Student {
    private String stuName;
    private String stuAge;

    public String getStuName() {
        return stuName;
    }

    public void setStuName(String stuName) {
        this.stuName = stuName;
    }

    public String getStuAge() {
        return stuAge;
    }

    public void setStuAge(String stuAge) {
        this.stuAge = stuAge;
    }
}
```

yml的配置：

```yml
person：
  name: zhangsan
  age: 18
  birth: 2017/04/21
  maps: {k1: v1,k2: v2}
  lists:
    - dog
    - cat
  student:
    name: lisi
    age: 14
```

其中map和list有两种配置方式

第一种：

```yaml
#map类型
maps:
  k1: v1
  k2: v2
#list类型
lists:
  - dog
  - cat
```

第二种行内设置：

```yaml
#map类型
maps: {k1: v1,k2: v2}
#list类型
lists: [dog,cat]
```

## 配置文件占位符

无论是在application.properties还是yml.properties中，都可以使用以下两种占位符

第一种：随机数占位符

```properties
${random.value} # 随机一串数，例如：821910b2a528f7ee35ea94fcc8682c01

${random.int}  # 随机整数，例如：-198290765

${random.int(max)} # 随机数小于max

${random.int(min,max)} # 随机数介于min和max之间,大于等于min，小于max

${random.long} # 随机长整型,例如：-7439395970807654149

${random.long(max)}

${random.long(min,max)}

${random.uuid} # 随机uuid，例如：561ec89a-5441-4211-84de-8a3ebaacf2e0
```

例子如下：

```yml
#该person就是上文prefix指定的值
person:
  first-name: zhang${random.value}  #字符串会和随机数直接拼接
  age: ${random.int(1,10)}
  birth: 2017/04/21
  maps: {k1: v1,k2: v2}
  lists:
    - dog
    - cat
  student:
    stuName: lisi
    stuAge: 20
```

第二种：属性配置占位符

属性配置占位符比较简单，举例如下：

```yml
#该person就是上文prefix指定的值
person:
  first-name: zhang
  age: 19
  birth: 2017/04/21
  maps: {k1: v1,k2: v2}
  lists:
    - dog
    - cat
  student:
    stuName: ${person.first-name}三      # 结果：zhang三
    stuAge: ${person.age}                # 结果：19
```

## 注解

#### WEB相关注解

```java
package com.hui.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@ResponseBody
@Controller
public class HelloController {

    @RequestMapping("/hello")
    public String sayHello(){
        return "hello";
    }
}
```

@Controller:将视图层注册到Spring容器中

@ResponseBody：将方法返回值返回给浏览器，让浏览器能够显示，也可以作用于方法上

@RequestMapping("/hello")：浏览器请求时的路径

> 其中注解@RestController=@Controller+@ResponseBody，所以也可以写成

```java
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String sayHello(){
        return "hello";
    }
}
```

#### ConfigurationProperties注解

作用：将配置文件(properties和yml)中配置的每一个属性的值，映射到这个组件中

@ConfigurationProperties：告诉springboot将本类中的所有属性和配置文件中相关的配置进行绑定

* prefix = "persion"：执行配置文件中"person"下面的所有属性一一映射

> 注意：只有这个组件是容器中的组件，才能使用容器提供的ConfigurationProperties注解功能

```java
package com.hui.pojo;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.Date;
import java.util.List;
import java.util.Map;


@ConfigurationProperties(prefix = "person")
@Component
public class Person {

    private String firstName;
    private String age;
    private Date birth;
    private Map<String,Object> maps;
    private List<String> lists;
    private Student student;

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getAge() {
        return age;
    }

    public void setAge(String age) {
        this.age = age;
    }

    public Date getBirth() {
        return birth;
    }

    public void setBirth(Date birth) {
        this.birth = birth;
    }

    public Map<String, Object> getMaps() {
        return maps;
    }

    public void setMaps(Map<String, Object> maps) {
        this.maps = maps;
    }

    public List<String> getLists() {
        return lists;
    }

    public void setLists(List<String> lists) {
        this.lists = lists;
    }

    public Student getStudent() {
        return student;
    }

    public void setStudent(Student student) {
        this.student = student;
    }

    @Override
    public String toString() {
        return "Person{" +
                "firstName='" + firstName + '\'' +
                ", age='" + age + '\'' +
                ", birth=" + birth +
                ", maps=" + maps +
                ", lists=" + lists +
                ", student=" + student +
                '}';
    }
}
```

```java
package com.hui.pojo;

public class Student {
    private String stuName;
    private String stuAge;

    public String getStuName() {
        return stuName;
    }

    public void setStuName(String stuName) {
        this.stuName = stuName;
    }

    public String getStuAge() {
        return stuAge;
    }

    public void setStuAge(String stuAge) {
        this.stuAge = stuAge;
    }

    @Override
    public String toString() {
        return "Student{" +
                "stuName='" + stuName + '\'' +
                ", stuAge='" + stuAge + '\'' +
                '}';
    }
}
```

yml配置文件：application.yml（注意格式，格式出错会报错）

```yaml
#该person就是上文prefix指定的值
person:
  #此处是松散绑定，虽然firstName=first-name，在松散绑定中-n相当于N
  first-name: zhang
  age: 19
  birth: 2017/04/21
  maps: {k1: v1,k2: v2}
  lists:
    - dog
    - cat
  student:
    stuName: lisi
    stuAge: 20
```

properties配置：application.properties

```properties
person.name=zhangsan
person.age=18
person.birth=2017/04/21
person.maps.k1=v1
person.maps.k2=v2
person.lists=dog,cat
person.student.stuName=lisi
person.student.stuAge=14
```

引入了注解@ConfigurationProperties后，可以加一个以下依赖（也可以不加）:

```xml
<!--导入配置文件处理器，配置文件进行绑定就会有提示-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

该依赖的作用是在写好了一个pojo对象后，在配置文件中写里面相应的属性时，会有提示

![](01.png)

#### Value注解

作用：赋值

使用方式：

* Value="字面量"：直接给赋值
* Value="${key}": 从环境变量、配置文件中获取值
* Value="#{SpEL}"：可以使用SpEL表达式进行赋值

```java
@Value("li")                //Value="字面量"：直接给赋值
private String firstName;
@Value("${person.birth}")   //Value="${key}": 从环境变量、配置文件中获取值
private Date birth;
@Value("#{11*2}")           //Value="#{SpEL}"：可以使用SpEL表达式进行赋值
private String age;
```

#### ConfigurationProperties和Value比较

|                      | @ConfigurationProperties | @Value     |
| -------------------- | ------------------------ | ---------- |
| 功能                 | 批量注入配置文件中的属性 | 一个个指定 |
| 松散绑定（松散语法） | 支持                     | 不支持     |
| SpEL                 | 不支持                   | 支持       |
| JSR303数据校验       | 支持                     | 不支持     |
| 复杂类型封装         | 支持                     | 不支持     |

松散绑定：

```
在配置文件（properties和yml）中，以下3个写法是相等的
person.firstName       标准格式
person.first-name      -n等效于N
person.first_name      _n等效于N
```

JSR303数据校验：

> 参看文章《Validated注解使用.md》

#### PropertySource和ImportResource注解

**@PropertySource：用于加载指定配置文件**

加载单个指定配置文件：

```java
@PropertySource(value = "classpath:person.properties")
@ConfigurationProperties(prefix = "person")
@Component
@Validated
public class Person {
    //todo...
}
```

如此加载的话，配置信息就可以不写在application.properties或者application.yml文件中了，可以写在person.properties中

```properties
#该person就是上文prefix指定的值
person.firstName=wang
person.age=19
person.birth=2017/04/21
person.maps.k1=v1
person.maps.k2=v2
person.lists=dog,cat
person.student.stuName=wangwu
person.student.stuAge=25
```

加载多个指定配置文件：

```java
@PropertySource(value = {"classpath:person.properties","classpath:student.properties"})
@ConfigurationProperties(prefix = "person")
@Component
@Validated
public class Person {
    //todo...
}
```

person.properties文件

```properties
#该person就是上文prefix指定的值
person.firstName=wang
person.age=19
person.birth=2017/04/21
person.maps.k1=v1
person.maps.k2=v2
person.lists=dog,cat
```

student.properties文件

```properties
person.student.stuName=wangwu
person.student.stuAge=25
```

**@ImportResource：导入spring的配置文件，让配置文件里面的内容生效**

我们先在resources文件夹下写一个bean.xml

```xaml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="personMethod" class="com.hui.service.PersonService"></bean>

</beans>
```

再写一个PersonService服务com.hui.service.PersonService

```java
package com.hui.service;

public class PersonService {
    public String personMethod(){
        return "I am a person!";
    }
}
```

接下来开始测试：

第一种测试：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {
    
    @Test
    public void personMethodTest() {
        ApplicationContext ac=new ClassPathXmlApplicationContext("bean.xml");
        System.out.println(ac.containsBean("personMethod"));
        PersonService personService= (PersonService) ac.getBean("personMethod");
        System.out.println(personService.personMethod());
    }
}
```

打印结果为true和I am a person!，这说明我们在new ClassPathXmlApplicationContext("bean.xml")时便加载了spring容器，将personMethod存到到了容器中，此种获取方法我们不需要使用ImportResource注解

第二种测试：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    ApplicationContext ac;

    @Test
    public void personMethodTest() {
        System.out.println(ac.containsBean("personMethod"));
    }
}
```

打印结果为false，这说明通过这种方法我们无法加载bean.xml文件，这时我们如果还想加载bean.xml文件，需要在springboot的启动类上加上注解@ImportResource，并指定要导入的文件

```java
@ImportResource(value = {"classpath:bean.xml"})
@SpringBootApplication
public class SpringbootdemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootdemoApplication.class, args);
    }

}
```

此时再运行测试类，便可以打印出true了

> 该注解在springboot中并不常用，因为springboot一般使用配置类，而不使用spring类型的xml文件。配置类的使用参考spring中的《Spring的新注解.md》



#### 测试方面注解

Springboot测试方面相关的注解如下：

com.hui.SpringbootgeneratorApplicationTests

```java
package com.hui;

import com.hui.pojo.Person;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootgeneratorApplicationTests {

    @Autowired
    private Person person;

    @Test
    public void contextLoads() {
        System.out.println(person.toString());
    }
}
```

> Srpingboot的版本如果使用<version>2.2.16.RELEASE</version>，发现@Runwith报错，使用<version>2.1.13.RELEASE</version>不报错

## Profile

用于多文件的配置，可以使我们自由切换不同的springboot配置文件，文件名是：application-{profile}.properties/yml

**application.properties**

对于application.properties的配置文件，我们可以创建多个其它的配置文件，例如有以下三个环境：

application.properties           默认环境

```properties
server.port=8080
# 通过配置该值来决定使用哪一个配置文件
#spring.profiles.active=dev
```

application-dev.properties     开发环境

```properties
server.port=8081
```

applicatio-prod.properties     生产环境

```properties
server.port=8082
```

使用具体哪个环境，需要在application.properties 文件中进行配置`spring.profiles.active`的值：

* 如果不配置，默认使用application.properties配置文件，
* 如果配置``spring.profiles.active`=dev`,使用application-dev.properties配置文件
* 如果配置``spring.profiles.active`=prod`,使用application-prod.properties配置文件

**application.yml**

对于application.yml，不需要配置多个文件，yml支持多文档快方式，每个文档快用`---`隔开

```yml
server:
  port: 8083

# 通过这个配置决定激活哪一个文档快，如果不配置，默认激活第一个文档快
spring:
  profiles:
    active: prod

---
server:
  port: 8084

# 如果使用多文档快，这一步必须配置，否则多文档快不会生效，会造成默认激活最后一个文档快
spring:
  profiles: dev

---
server:
  port: 8085

# 如果使用多文档快，这一步必须配置，否则多文档快不会生效，会造成默认激活最后一个文档快
spring:
  profiles: prod

```

> 注意：如果使用多文档快，除了第一个文档快外的其他文档快都要配置上spring-profiles，否则会被默认激活最后一个文档快


