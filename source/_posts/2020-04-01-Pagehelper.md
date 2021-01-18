---
title: Pagehelper
date: 2020-04-01 20:47:08
categories:
- Pagehelper
tags:
- 分布式案例
- Pagehelper
---

<center><font size=4 color="red">Pagehelper的使用</font></center>

<!--more-->

## Pagehelper的使用

SqlMapConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <!--pageHelper版本5以前的配置-->
    <!--<plugins>-->
    <!--    <plugin interceptor="com.github.pagehelper.PageHelper">-->
    <!--        <property name="dialect" value="mysql"/>-->
    <!--    </plugin>-->
    <!--</plugins>-->

    <!--pageHelper版本5以后的配置-->
    <plugins>
        <!-- com.github.pagehelper为PageHelper类所在包名 -->
        <plugin interceptor="com.github.pagehelper.PageInterceptor">
            <!-- 设置数据库类型 Oracle,Mysql,MariaDB,SQLite,Hsqldb,PostgreSQL六种数据库-->
            <property name="helperDialect" value="mysql"/>
        </plugin>
    </plugins>
</configuration>
```

依赖jar包

```xml
<!--mybatis分页相关的-->
<dependency>
    <groupId>com.github.miemiedev</groupId>
    <artifactId>mybatis-paginator</artifactId>
    <version>1.2.15</version>
</dependency>
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>5.1.2</version>
</dependency>
```

设置分页信息

```java
//获取第1页，10条内容，默认查询总数count
Pagehelper.startPage(1,10);

//紧跟着的第一个select方法会被分页
List<User> list=userMapper.selectIf(1);
```

获取分页信息有两种方法

第一种方法

```java
//分页后，实际返回的结果list类型是Page<E>，如果想取出分页信息，需要强制转换为Page<E>
Page<User> userList=(Page<User>)list;
userList.getTotal();
```

第二种方法，**推荐使用第二种**

```java
//用PageInfo对结果进行包装
PageInfo page=new PageInfo(list);
//获取PageInfo全部属性，PageInfo包含了非常全面的分页属性
assertEquals(1,page.getPageNum());
assertEquals(10,page.getPageSize());
assertEquals(1,page.getStartRow());
assertEquals(10,page.getEndRow());
assertEquals(183,page.getTotal());
assertEquals(19,page.getPages());
assertEquals(1,page.getFirstPage());
assertEquals(8,page.getLastPage());
assertEquals(true,page.isFirstPage());
assertEquals(false,page.isLastPage());
assertEquals(true,page.isHasNextPage());
assertEquals(false,page.isHasPreviousPage());
```

java代码实践

```java
public class DemoTest {
    @Test
    public void testhelper() throws IOException {
        //初始化spring容器
        ApplicationContext context=new ClassPathXmlApplicationContext("classpath:spring/applicationContext.xml");
        //获取mapper代理对象
        UserMapper userMapper = context.getBean(UserMapper.class);

        //取出第一页，一页2个信息
        PageHelper.startPage(1,2);
        UserExample example = new UserExample();
		
        //去掉查询条件，是查询所有
		//UserExample.Criteria criteria = example.createCriteria();
		//criteria.andUseridEqualTo(1);
        List<User> users = userMapper.selectByExample(example);
        PageInfo<User> info=new PageInfo<User>(users);
        //Assert.assertEquals(info.getPages(),3,null)的使用，第一个参数是实际参数，第二个是期望参数，第3个是报错信息
        //Assert.assertEquals(info.getPages(),3,null);
        System.out.println(info.getTotal());
    }
}
```
