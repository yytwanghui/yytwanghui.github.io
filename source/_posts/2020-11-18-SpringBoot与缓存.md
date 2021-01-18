---
title: 09_SpringBoot与缓存
date: 2020-11-18 19:07:45
categories:
- SpringBoot
tags:
- SpringBoot
- 缓存
---

<center><font size=4 color="red">09_SpringBoot与缓存</font></center>

<!--more-->

# SpringBoot与缓存

## JSR107规范

针对缓存所制定的规范，因为比较复杂，所以基本不会使用，这里不再讲解

## Spring缓存抽象

目的：简化缓存开发，并且支持JSR107缓存注解

#### 几个重要概念&缓存注解

| 概念&注解      | 功能                                                         |
| -------------- | ------------------------------------------------------------ |
| Cache          | 缓存接口，定义缓存操作，实现有：RedisCache、EhCacheCache、ConcurrentMapCache等 |
| CacheManager   | 缓存管理器，管理各种缓存(Cache)组件                          |
| @Cacheable     | 主要针对方法配置，根据方法的请求参数对返回结果进行缓存       |
| @CacheEvict    | 清空缓存，主要用于删除的方法上，在删除数据时同时删除缓存数据 |
| @CachePut      | 更新缓存，主要用于更新的方法上，在更新数据时同时更新缓存数据 |
| @EnableCaching | 开启基于注解的缓存                                           |
| keyGenerator   | 缓存数据时key生成策略                                        |
| serialize      | 缓存数据时value序列化策略                                    |

如果要使用缓存，首先需要在启动类上加上注解：**@EnableCaching**

为了讲解属性时，我们能清晰的看到sql的打印结果，我们使用以下logback-spring.xml的配置：

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
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>

    <logger name="com.hui.dao" level="DEBUG" additivity="false">
        <appender-ref ref="STDOUT" />
    </logger>

</configuration>
```

#### Cache SpEL

作用，在@Cacheable注解下的有些属性是支持SpEL表达式的

| 名字          | 位置               | 描述                   | 使用（获取）示例           |
| ------------- | ------------------ | ---------------------- | -------------------------- |
| methodName    | root object        | 用于做缓存的方法名     | #root.methodName           |
| method        | root object        | 用于做缓存的方法       | #root.method.name          |
| target        | root object        | 当前被调用的目标对象   | #root.target               |
| targetClass   | root object        | 当前被调用的目标对象类 | #root.targetClass          |
| args          | root object        | 当前被调用的方法列表   | #root.args[0]              |
| caches        | root object        | 缓存组件名称列表       | #root.caches[0].name       |
| argument name | evaluation context | 方法参数的名字         | #a0 或者 #p0 （0表示索引） |
| result        | evaluation context | 方法执行后的返回值     | #result                    |

对caches进行解释一下，例如

```
@Cacheable(value={"user","users"})
使用：#root.caches[0].name获取到的值就是user
使用：#root.caches[1].name获取到的值就是users
```

另外：  #root.args[0] = #a0 =#p0 都表示第一个参数的值

#### @Cacheable

作用：将方法运行结果进行缓存，以后再要相同的数据时，直接从缓存中获取，不用再调用方法

步骤：先看缓存中有没有指定的key，如果有，就不再调方法了，直接从缓存中获取

一个缓存的例子：

```java
//查
@Cacheable(cacheNames = "user")
public User findUser(Integer userId){
    return userMapper.selectByPrimaryKey(userId);
}
```

其下的属性：

* cacheNames/value：CacheManager是缓存管理器，管理着其下的缓存组件Cache，而每一个缓存组件都有一个名称。属性cacheNames/value就是定义组件名称的。是数组的方式，可以指定在多个缓存中，例如：

  ```
  cacheNames = {"user","users"}
  ```

* key：就是缓存key-value中需要指定的key，默认是使用方法参数的值，如果不指定，在上面的缓存例子中，默认key的值就是userId对应的值，如果userId=1，那key的值就是1。例外key的值支持编写SPEL表达式，例如`key = "#userId"`也是指定key的值是userId对应的值

* keyGenerator：key的生成器，其实就是指定key是如何生成的，可以自定义key的生成策略，属性key和keyGenerator只能使用一个，不能同时存在

* cacheManager：指定缓存管理器，可以指定由redis、EhCache或concurrentMap等管理缓存，默认是使用concurrentMapCacheManager管理缓存。或者由cacheResolver指定获取解析器。cacheManager和cacheResolver不能同时存在

* condition：满足指定条件是才进行缓存。例如：`condition = "#userId>1"`表示userId的值大于1时才进行缓存，其也可以使用SPEL表达式

* unless：否定缓存，当unless指定的条件成立时，就不进行缓存。例如：`unless = "#userId==1"`表示userId的值等于1时，就不进行缓存。这个属性也可以对结果进行判断，`#result`就是获取结果，例如：`unless = "#result==null"`表示缓存的value值如果为null，就不进行缓存

* sync：是否开启异步模式

#### 缓存原理分析

关于缓存的自动配置类：CacheAutoConfiguration

在该类中我们可以通过`@Import(CacheConfigurationImportSelector.class)`查看导入了什么组件

```java
static class CacheConfigurationImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        CacheType[] types = CacheType.values();
        String[] imports = new String[types.length];
        for (int i = 0; i < types.length; i++) {
            imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
        }
        //imports就是导入的所有组件,在该处打断点启动项目，可以在控制台看到都有哪些组件
        return imports;
    }

}
```

我们通过打断点的方式启动项目，可以看到imports里有以下10个组件：

```powershell
org.springframework.boot.autoconfigure.cache.GenericCacheConfiguration
org.springframework.boot.autoconfigure.cache.JCacheCacheConfiguration
org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration
org.springframework.boot.autoconfigure.cache.HazelcastCacheConfiguration
org.springframework.boot.autoconfigure.cache.InfinispanCacheConfiguration
org.springframework.boot.autoconfigure.cache.CouchbaseCacheConfiguration
org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration
org.springframework.boot.autoconfigure.cache.CaffeineCacheConfiguration
org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration
org.springframework.boot.autoconfigure.cache.NoOpCacheConfiguration
```

以dubug的模式启动Springboot项目，查看上面的缓存组件中哪些组件生效，经过查找，发现只有组件`org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration`生效

```java
@Bean
public ConcurrentMapCacheManager cacheManager() {
    ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
    List<String> cacheNames = this.cacheProperties.getCacheNames();
    if (!cacheNames.isEmpty()) {
        cacheManager.setCacheNames(cacheNames);
    }
    return this.customizerInvoker.customize(cacheManager);
}
```

看到SimpleCacheConfiguration类的作用是给容器中放入一个CacheManager：ConcurrentMapCacheManager

进入`ConcurrentMapCacheManager`类

```java
@Nullable
public Cache getCache(String name) {
    Cache cache = (Cache)this.cacheMap.get(name);
    if (cache == null && this.dynamic) {
        synchronized(this.cacheMap) {
            cache = (Cache)this.cacheMap.get(name);
            if (cache == null) {
                cache = this.createConcurrentMapCache(name);
                this.cacheMap.put(name, cache);
            }
        }
    }

    return cache;
}
```

可以看到`ConcurrentMapCacheManager`类的作用有：获取或者创造`ConcurrentMapCache`

进入`ConcurrentMapCache`类，可以看到：

```java
@Nullable
protected Object lookup(Object key) {
    return this.store.get(key);
}
```

从store查询缓存组件，store就是ConcurrentMap

所以`ConcurrentMapCache`的作用就是将缓存数据保存在`ConcurrentMap`中

#### @Cacheable运行流程

1、方法运行之前，先去查询Cache（缓存组件），按照cacheNames指定的名字获取；

2、去Cache中查找缓存的内容，使用一个key，默认就是方法的参数；key是按照某种策略生成的；是使用keyGenerator生成的，默认的keyGenerator是SimpleKeyGenerator

SimpleKeyGenerator生成key的策略：

* 如果没有参数，默认key=new SimpleKey();  测试结果`key=SimpleKey []`
* 如果有一个参数，key=参数的值
* 如果有多个参数，key=new SimpleKey(params);

> 不管有没有参数，有几个参数，我们都可以通过key来自己制定key的值

3、没有查到缓存就调用目标方法

4、将目标方法返回的结果放到缓存中

**总的来说：**

@Cacheable标注的方法执行之前先来检查缓存中有没有这个数据，默认按照参数的值作为key去查询缓存

如果没有数据，运行方法，将结果放入缓存；如果有数据，直接从缓存中获取

**核心：**

* 使用CacheManager【默认是ConcurrentMapCacheManager】按照名字获取Cache【默认是ConcurrentMapCache】组件
* key使用keyGenerator【默认是SimpleKeyGenerator】生成的

#### @Cacheable缓存下的几个属性

**key/keyGenerator**

作用：用于生成缓存的key，不能同时使用

key：可以使用SpEL表达式，例如：

```java
//查
@Cacheable(cacheNames = {"user"},key = "#root.methodName+'['+ #userId+']'")
public User findUser(Integer userId){

    return userMapper.selectByPrimaryKey(userId);
}
```

如果userId=2，这时缓存中存的key就是：findUser[2]

> 注意：这个值可以在类CacheAspectSupport.class下的findCachedItem方法中对获取的key打断点查看

keyGenerator：指定key的生成策略，我们可以自定义其生成策略

自定义生成策略的配置类：

```java
package com.hui.config;

import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.lang.reflect.Method;
import java.util.Arrays;

@Configuration
public class MyKeyGenerator  {

    @Bean("mySelfKeyGenerator")
    //修改缓存中key的生成策略
    public KeyGenerator keyGenerator(){
        return new KeyGenerator() {
            @Override
            public Object generate(Object o, Method method, Object... objects) {
                //return method.getName()+"["+ Arrays.asList(objects).toString()+"]";
                
                //以类名+方法名作为key的值,建议使用这种生成策略，可以避免因为参数的不同对key产生不一样的				 影响，具体采用哪种生成策略，还要根据具体项目决定
                String[] strings = o.getClass().getName().split("\\.");
                //key=UserService-findUser
                return strings[strings.length-1]+"-"+method.getName();
            }
        };
    }
}
```

使用我们自定义的keyGenerator

```java
//查
@Cacheable(cacheNames = {"user"},keyGenerator = "mySelfKeyGenerator")
public User findUser(Integer userId){

    return userMapper.selectByPrimaryKey(userId);
}
```

结果：如果userId=2，保存的缓存key的值是：findUser[[2]]，这里有两个[]，外面是是我们拼接的，里面是数组转成toString时自己带上的

**condition/unless**

上面介绍的已经比较详细了，这里不再介绍了。只是condition如果要使用多个条件是，可以用and进行连接

```java
@Cacheable(cacheNames = {"user"},condition = "#userId==2 and #root.methodName eq 'findUser'")
public User findUser(Integer userId){
    return userMapper.selectByPrimaryKey(userId);
}
```

#### 缓存数据的获取

对于一个新的方法或者新的类中的方法，如何获取已经存入到缓存中的数据呢？可以参考以下实例：

```java
//查询
@Cacheable(cacheNames = {"user"},key = "'cache'+#userId")
public User findUser(Integer userId){
    return userMapper.selectByPrimaryKey(userId);
}


//新的查询，使用和findUser一样的key
@Cacheable(cacheNames = {"user"},key = "'cache'+#userId")
public User findUsers(Integer userId){
    //执行步骤是，先判断key=cache+userId的值在缓存中有没有，如果有，下面的return方法不再执行，并把value的值返回
    //如果没有，就去执行查询操作，并把新的key=cache+userId的值和value存入缓存中
    return userMapper.selectByPrimaryKey(userId);
}
```

#### @CachePut

作用：调用方法，并把更新后的数据存入到缓存中

步骤：先调方法，再把更新后的数据存入到缓存中，方法是先执行的

com.hui.service.UserService内方法：

```java
//查询
@Cacheable(cacheNames = {"user"},key = "'cache'+#userId")
public User findUser(Integer userId){
    return userMapper.selectByPrimaryKey(userId);
}

//更新
@CachePut(cacheNames = "user",key = "'cache'+#result.userId")
public User updateUser(User user){
    userMapper.updateByPrimaryKey(user);
    return user;
}
```

com.hui.controller.UserController内方法：

```java
//更新
@RequestMapping(value = "/add", method= RequestMethod.POST)
public User updateUser( @RequestBody User user) {
    return userService.updateUser(user);
}
//查询
@GetMapping("/users/{userId}")
public User findUsers(@PathVariable(value = "userId") Integer userId) {
    return userService.findUsers(userId);
}
```

测试步骤：

1. 先调用：http://localhost:8080/user/1进行查询，去数据库查询得到数据，并把userId=1查询到的结果放到缓存中，缓存中的key=cache1

2. 再调用http://localhost:8080/add更新数据，随便更新以下的某个字段，将数据库中的数据进行更新，并将缓存中的数据更新

   ```json
   {
       "userId": 1,
       "userName": "xiaohui",
       "userAge": 25,
       "date": "2020-08-26T00:00:00.000+0000"
   }
   ```

3. 再调用http://localhost:8080/user/1进行查询，可以直接从缓存中拿到更新后的数据，不需要去查询数据库

> 注意：如果执行第3步保证能获取到更新后的缓存数据，前提是：@CachePut(cacheNames = "user",key = "'cache'+#result.userId")中的cacheNames 和key的值必须是一样的，如果不一样，则会放到不同的key-value中，只有一样，才能保证，更新后的缓存数据覆盖掉以前的缓存数据

#### @CacheEvict

作用：在执行删除操作时，同时删除缓存

步骤：可以在方法执行前删除缓存，也可以在方法执行后删除缓存，由属性beforeInvocation控制

```java
//删，先执行方法，根据userId删除数据库数据，然后执行缓存删除，根据key='cache'+#userId删除执行缓存
@CacheEvict(cacheNames = "user",key = "'cache'+#userId")
public void deleteUser(Integer userId){
    userMapper.deleteByPrimaryKey(userId);
}
```

**属性：allEntries**

默认：allEntries = false，当另allEntries = true时表示删除cacheNames = "user"的所有缓存数据

```java
//删
@CacheEvict(cacheNames = "user",allEntries = true)
public void deleteUser(Integer userId){
    userMapper.deleteByPrimaryKey(userId);
}
```

**属性：beforeInvocation**

默认：beforeInvocation = false，表示先执行方法，再执行删除缓存操作

如果：beforeInvocation = true，表示先删除缓存，再执行方法，这样设置可以保证如果方法中有异常无法执行，也能先保证把缓存数据给删除

```java
//删
@CacheEvict(cacheNames = "user",beforeInvocation = true)
public void deleteUser(Integer userId){
    userMapper.deleteByPrimaryKey(userId);
}
```

#### @Caching

作用：用于定于复杂缓存注解

```java
public @interface Caching {
    Cacheable[] cacheable() default {};

    CachePut[] put() default {};

    CacheEvict[] evict() default {};
}
```

从接口可以看到被@Caching注解的缓存可以同时使用里面的Cacheable、CachePut、CacheEvict，并且都是数组的形式

数据库中数据：

| userId | userName | userAge | date                       |
| ------ | -------- | ------- | -------------------------- |
| 1      | xiaohui  | 25      | 2020-08-26 08:00:00.000000 |

service的类：com.hui.service.UserService

```java
//查询,标记复杂注解的查询，根据userId查询
@Caching(
    cacheable = {
        //根据key=userId的值查询缓存，如果没有查到，就放到缓存中
        @Cacheable(cacheNames = "user",key = "#userId")
    },
    put = {
        //查询结束后，将根据key=userId的值更新缓存
        @CachePut(cacheNames = "user",key = "#result.userId"),
        //查询结束后，将根据key=userName的值更新缓存
        @CachePut(cacheNames = "user",key = "#result.userName"),
    }
)
public User findUser(Integer userId){
    return userMapper.selectByPrimaryKey(userId);
}

//标记@Cacheable注解的查询，根据userName查询
@Cacheable(cacheNames = "user",key = "#userName")
public User findUserByName(String userName){
    UserExample example=new UserExample();
    UserExample.Criteria criteria = example.createCriteria();
    criteria.andUserNameEqualTo(userName);
    return userMapper.selectByExample(example).get(0);
}
```

测试类：com.hui.controller.UserController

```java
@GetMapping("/user/{userId}")
public User findUser(@PathVariable(value = "userId") Integer userId) {
    return userService.findUser(userId);
}

@GetMapping("/username/{userName}")
public User findUserByName(@PathVariable(value = "userName") String userName) {
    return userService.findUserByName(userName);
}
```

测试步骤：

1. 访问http://localhost:8080/user/1，这里会调用数据库查询出数据，并根据key=userId将数据放入到缓存中，同时根据key=userId和userName分别将数据更新到缓存中，这时缓存中有的数据有两套key-value，分别是：key=userId以及对应的value；key=userName以及对应的value。value就是从数据库查到的值
2. 这时我们访问http://localhost:8080/userName/xiaohui，因为缓存中有key=xiaohui的数据，所以不需要再查数据库，可以直接从缓存中获取
3. 但是如果再访问http://localhost:8080/user/1，发现并没有从缓存中获取key=1的数据，而是又查了数据库，原因是复杂注解@Caching中包含了@CachePut注解，而我们访问http://localhost:8080/user/1最终调用的是service层中的findUser方法，因为有@CachePut注解，方法就一定会优先执行，所以会再次调用数据库，而后才从缓存中获取。

#### @CacheConfig

作用：抽取公共的注解属性，作用在类上

例如上面我们每个方法上的缓存注解都有属性cacheNames，我们就可以通过注解@CacheConfig将该属性抽取出来，以后在方法的缓存注解里，就不用再调价该属性了。

```java
@Service
@CacheConfig(cacheNames = "user")
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @CacheEvict(key = "'cache'+#userId")
    public void deleteUser(Integer userId){
        userMapper.deleteByPrimaryKey(userId);
    }

    @Cacheable(key = "#userName")
    public User findUserByName(String userName){
        UserExample example=new UserExample();
        UserExample.Criteria criteria = example.createCriteria();
        criteria.andUserNameEqualTo(userName);
        return userMapper.selectByExample(example).get(0);
    }
}
```

@CacheConfig注解可抽取的其它属性有以下4个：

 ```java
public @interface CacheConfig {
    String[] cacheNames() default {};

    String keyGenerator() default "";

    String cacheManager() default "";

    String cacheResolver() default "";
}
 ```

## Docker-Compose安装Redis

docker-compose.yml文件：

```yaml
version: '3'

services:
  redis:
    image: redis
    container_name: redis
    hostname: redis
    restart: always
    ports:
      - 6379:6379
    networks:
      - net_db
    volumes:
      - ./conf/redis.conf:/etc/redis/redis.conf:rw
      - ./data:/data:rw
    command:
      # 设置持久化方式和redis的访问密码
      redis-server /etc/redis/redis.conf --appendonly yes --requirepass 123456

networks:
  net_db:
    driver: bridge
```

## Springboot使用Redis

引入pom依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

添加application.yml配置：

```yaml
spring:  
  # redis相关配置
  redis:
    host: 192.168.31.180
    port: 6379
    password: 123456
    jedis:
      pool:
        max-active: 100
        max-idle: 10
        max-wait: 6000
    timeout: 3000
```

**特别注意：**

我们在引入Redis的组件后，Springboot在启动缓存组件时启动的是

```
org.springframework.boot.autoconfigure.cache.RedisCacheConfiguration
```

而基础缓存组件`org.springframework.boot.autoconfigure.cache.SimpleCacheConfiguration`不再匹配，而我们使用的缓存管理器是：RedisCacheManager，这个缓存管理器创造的是：RedisCache来管理缓存。此时我们再使用缓存相关的注解时，我们的缓存数据会存到redis中

简单的测试：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    //StringRedisTemplate针对字符，保存的缓存数据key和value都是String
    private StringRedisTemplate stringRedisTemplate;

    @Autowired
     //RedisTemplate保存的缓存数据key是String，value是object，在做缓存对象时，可以用这个
    private RedisTemplate redisTemplate;

    @Test
    public void cacheString() {
        stringRedisTemplate.opsForValue().append("name","zhangsan");
    }

    @Test
    public void cacheObject() {
        //先从数据库查询到user对象
        User user = userMapper.selectByPrimaryKey(1);
        //将对象保存到redis中
        redisTemplate.opsForValue().set("user",user);
    }

}
```

我们这样虽然能够把对象数据存到redis中，但是因为存入的过程使用的序列化是jdk方式的序列化，所以我们在redis中无法看到具体的实例，但是可以通过redisTemplate.opsForValue().get("user");获取到redis中的数据，但是我们也可以修改存入redis中的序列化方式，让其以json的方式序列化存入到redis中

**序列化配置文件**

com.hui.config.MyRedisConfig

```java
package com.hui.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;


@Configuration
public class MyRedisConfig {

    /**
     * redisTemplate 序列化使用的jdkSerializeable, 存储二进制字节码, 所以自定义序列化类
     * 修改序列化方式为json：Jackson2JsonRedisSerializer
     * @param redisConnectionFactory
     * @return
     */
    // 注入 Spring 容器中，忽略所有警告
    @Bean(name = "myRedisTemplate")
    @SuppressWarnings("all")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        // key 采用 String 的序列化方式
        template.setKeySerializer(stringRedisSerializer);
        // hash 的 key 也采用 String 的序列化方式
        template.setHashKeySerializer(stringRedisSerializer);
        // value 的序列化方式采用 JSON
        template.setValueSerializer(jackson2JsonRedisSerializer);
        // hash value 的序列化方式也采用 JSON
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```

使用我们自己序列化的文件

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootdemoApplicationTests {
    
    @Autowired
    @Qualifier("myRedisTemplate")
    private RedisTemplate<String, Object> userRedisTemplate;
    
    @Test
    public void cacheObject() {
        //先从数据库查询到user对象
        User user = userMapper.selectByPrimaryKey(1);
        System.out.println(user.toString());
        //将对象保存到redis中
        userRedisTemplate.opsForValue().set("user",user);
    }

}    
```

这样我们存入到redis中的key是以String存入，value的值是以json的方式进行存入

#### StringRedisTemplate的使用

```java
stringRedisTemplate.opsForValue().set("test", "100",60*10,TimeUnit.SECONDS);//向redis里存入数据和设置缓存时间  

stringRedisTemplate.boundValueOps("test").increment(-1);//val做-1操作  

stringRedisTemplate.opsForValue().get("test")//根据key获取缓存中的val  
    
stringRedisTemplate.boundValueOps("test").increment(1);//val +1  

stringRedisTemplate.getExpire("test")//根据key获取过期时间  
    
stringRedisTemplate.getExpire("test",TimeUnit.SECONDS)//根据key获取过期时间并换算成指定单位  
    
stringRedisTemplate.delete("test");//根据key删除缓存 

stringRedisTemplate.hasKey("546545");//检查key是否存在，返回boolean值  

stringRedisTemplate.opsForSet().add("red_123", "1","2","3");//向指定key中存放set集合  

stringRedisTemplate.expire("red_123",1000 , TimeUnit.MILLISECONDS);//设置过期时间 

stringRedisTemplate.opsForSet().isMember("red_123", "1")//根据key查看集合中是否存在指定数据  
    
stringRedisTemplate.opsForSet().members("red_123");//根据key获取set集合 
```

## 缓存抽象将数据存入redis

缓存注解主要包括注解：

```
@Cacheable
@CachePut
@CacheEvict
```

当我们使用了`spring-boot-starter-data-redis`依赖后，使用缓存注解的数据都缓存到redis中了，所以这里我们在注解上无需做任何改变，但是被**缓存注解**所注解的数据缓存到redis中序列化方式依然是jdk的序列化方式，因此，我们需要自己定义**RedisCacheManager**组件来修改**缓存注解**将数据缓存到redis的序列化方式，修改如下：

```java
package com.hui.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;


@Configuration
public class MyRedisConfig {

    private Duration timeToLive = Duration.ZERO;

    public void setTimeToLive(Duration timeToLive) {
        this.timeToLive = timeToLive;
    }


    /**
     * 自定义RedisCaCheManager，来解决序列化的问题，可以保证通过缓存注解存入到redis中的数据按照自己定义的序列化方式保存
     * 常用缓存注解：
     *  @Cacheable
     *  @CachePut
     *  @CacheEvict
     * @param factory
     * @return
     */
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisSerializer<String> stringRedisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);

        //解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        // 配置序列化
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(timeToLive)
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(stringRedisSerializer))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
                .disableCachingNullValues();

        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
        return cacheManager;

    }
}
```

**注意：**通过RedisCaCheManager存入到redis的key都是：cacheNames的值+：：+指定的key的值

例如：userId=1

```
@Cacheable(cacheNames = "user",key = "#userId"）
生成的key是：user::1
```

如果用的上面自定义的keyGenerator生成策略

```
@Cacheable(cacheNames = "user",keyGenerator = "mySelfKeyGenerator")
生成的key是：user::findUser[[1]]
```

如果想改变默认生成的key的值，可以对自定义的RedisCaCheManager做一下配置：

```java
/**
     * 自定义RedisCaCheManager，来解决序列化的问题，可以保证通过缓存注解存入到redis中的数据按照自己定义的序列化方式保存
     * 常用缓存注解：
     *  @Cacheable
     *  @CachePut
     *  @CacheEvict
     * @param factory
     * @return
     */
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisSerializer<String> stringRedisSerializer = new StringRedisSerializer();
    Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);

    //解决查询缓存转换异常的问题
    ObjectMapper om = new ObjectMapper();
    om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
    om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
    jackson2JsonRedisSerializer.setObjectMapper(om);

    // 配置序列化
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(timeToLive)
        .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(stringRedisSerializer))
        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
        //修改key的前缀
        .computePrefixWith(cacheName -> "")
        .disableCachingNullValues();

    RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .build();
    return cacheManager;

}
```

上面只是加了一个：

```java
//cacheName -> "" 的意思是：改变cacheName的前缀名为空，前缀的值就是->后面的值
.computePrefixWith(cacheName -> "")
```

#### RedisTemplate+缓存注解序列化

这是需要注意的一点：我们使用上面自定义的RedisTemplate来改变RedisTemplate方法将数据存入到redis的序列化方式，通过自定义RedisCacheManager来改变缓存注解将数据存入到redis的序列化方式。因此RedisTemplate+缓存注解序列化就是将上面两个自定义方法组合起来而已

com.hui.config.MyRedisConfig

```java
package com.hui.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;


@Configuration
public class MyRedisConfig {

    /**
     * 自定义RedisTemplate模板，可以保证我们在使用RedisTemplate来保存数据到redis是以自定义的序列化进行保存到redis
     * @param redisConnectionFactory
     * @return
     */
    // 注入 Spring 容器中，忽略所有警告
    @Bean(name = "myRedisTemplate")
    @SuppressWarnings("all")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        // key 采用 String 的序列化方式
        template.setKeySerializer(stringRedisSerializer);
        // hash 的 key 也采用 String 的序列化方式
        template.setHashKeySerializer(stringRedisSerializer);
        // value 的序列化方式采用 JSON
        template.setValueSerializer(jackson2JsonRedisSerializer);
        // hash value 的序列化方式也采用 JSON
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }


    private Duration timeToLive = Duration.ZERO;

    public void setTimeToLive(Duration timeToLive) {
        this.timeToLive = timeToLive;
    }


    /**
     * 自定义RedisCaCheManager，来解决序列化的问题，可以保证通过缓存注解存入到redis中的数据按照自己定义的序列化方式保存
     * 常用缓存注解：
     *  @Cacheable
     *  @CachePut
     *  @CacheEvict
     * @param factory
     * @return
     */
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisSerializer<String> stringRedisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);

        //解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        // 配置序列化（解决乱码的问题）
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(timeToLive)
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(stringRedisSerializer))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
                .disableCachingNullValues();

        RedisCacheManager cacheManager = RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
        return cacheManager;

    }
}
```
