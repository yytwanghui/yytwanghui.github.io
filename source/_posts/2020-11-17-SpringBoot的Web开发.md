---
title: 01_SpringBoot的Web开发
date: 2020-11-17 17:45:54
categories:
- SpringBoot
tags:
- SpringBoot
- Web
---

<center><font size=4 color="red">01_SpringBoot的Web开发</font></center>

<!--more-->

# Web开发

关于Web的配置，大多数都在WebMvcAutoConfiguration中

## 引入静态资源

做web开发，首先要引入静态资源，静态资源分为：

* 别人写好的库，例如：jQuery
* 我们自己写的静态资源，例如：我们写的css、js、html、图片等
* 工程首页
* favicon.ico的网页小图标

首尔我们要知道这些静态资源的路径如何放置

#### 别人写好的库

例如jQuery，不再是导入js文件，而是导入jQuery的jar包

从org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration.java类中我们可以看到以下方法：

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
    CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
                                             .addResourceLocations("classpath:/META-INF/resources/webjars/")
                                             .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
                                             .addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
                                             .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
}
```

我们导入的所有webjars包下的文件，就是我们导入的别人写好的库，其资源就在classpath:/META-INF/resources/webjars/下,其中这里的classpath是导入jar包的路径

以maven依赖的形式导入webjars包：[官网](https://www.webjars.org/)

![](webjars.png)

```xml
<dependency>
    <groupId>org.webjars.bower</groupId>
    <artifactId>jquery</artifactId>
    <version>3.5.1</version>
</dependency>
```

#### 我们自己写的静态资源

同样的从addResourceHandlers方法可以看到，自己写的静态资源从resourceProperties获取

```java
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/",
			"classpath:/resources/", "classpath:/static/", "classpath:/public/" };
```

可以知道自己写的静态资源是从resources下的：

* /META-INF/resources/
* /resources/
* /static/
* /public/

中获取的，所以我们写的静态资源可以放到以上4个静态资源文件夹下

#### 工程首页

工程首页的路径直接放在/META-INF/resources/，/resources/， /static/，/public/下，但是名称必须命名为**index.html**

```java
private Resource getIndexHtml(String location) {
    return this.resourceLoader.getResource(location + "index.html");
}
```

访问时直接：localhost:8080

#### favicon.ico的网页小图标

和工程首页一样，直接放到/META-INF/resources/，/resources/， /static/，/public/下即可

名称要是**favicon.ico**

## thymeleaf

作用：Springboot中不使用jsp，而是使用的thymeleaf

导入jar包：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

thymeleaf这里不再讨论，因为后期我并不打算使用

## WEB扩展与全面接管

SpringBoot提供了web的自动配置，但是一些mvc的其它配置功能并没有提供，这时需要扩展

例如：

 ```xml
<!-- 配置跳转，在访问/hello时，跳转到success.html页面-->
<mvc:view-controller path="/hello" view-name="success"/>

<!-- 配置拦截器，拦截/hello,拦截的bean是XXX-->
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/hello"/>
        <bean class="XXX"></bean>
    </mvc:interceptor>
</mvc:interceptors>
 ```

如果要实现以上的xml配置，可以使用配置类扩展如下：

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    public void addViewControllers(ViewControllerRegistry registry){
        //这句话的意思是即使访问/wanghui,也转发到/hello
        registry.addViewController("/wanghui").setViewName("/hello");
    }
}
```

这里的方法不能加@bean注解，因为是扩展，是在原有自动配置上再增加的功能，这里我们这样写后，springboot不仅仅把自动配置的addViewControllers方法加载进去，而且还把我们自己写的addViewControllers方法加载进去。

如果要全面托管SpringMvc的自动配置，在我们自己的配置类上加上注解：@EnableWebMvc，这样关于Web的自动配置便会失效，只有我们配置类中使用的才会生效，不推荐使用

#### 引入资源

引入资源不再介绍，后续应该不会使用thymeleaf

#### 国际化

SpringMvc实现国际化的步骤：

1. 编写国际化配置文件
2. 使用ResourceBundleMessageSource管理国际化资源文件
3. 在页面使用fmt:message取出国际化内容

在Springboot中因为有了自动配置，所以只需要编写国际化配置文件即可，其实现国际化的方式，以登录为例

1. 先在resources下创建一个包i18n

2. 创建一个登录的默认配置文件：login.propreties

3. 创建一个登录的中文格式的配置文件：login_zh_CN.propreties，这一步添加完，视图会自动变换为国际化视图

4. 添加英文的国际化配置文件

   ![](国际化视图.jpg)

   ![](英文国际化.jpg)

   我们就得到了login_en_US.propreties

5. 添加需要国际化的配置信息

   ![](配置国际化.jpg)

   login.tip就是要国际化的属性，一般都会有多个属性，例如：登录、注册、确认等等，这时就逐个添加就行了

   ![](国际化配置2.jpg)

   切回Text视图就可以看到已经配置好的属性内容了

   ![](国际化配置完成.jpg)

6. Springboot已经给我们自动配置了国际化，自动配置类为：MessageSourceAutoConfiguration

   ```java
   @EnableConfigurationProperties
   public class MessageSourceAutoConfiguration {
       @Bean
   	@ConfigurationProperties(prefix = "spring.messages")
   	public MessageSourceProperties messageSourceProperties() {
   		return new MessageSourceProperties();
   	}
   
   	@Bean
   	public MessageSource messageSource(MessageSourceProperties properties) {
   		ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
   		if (StringUtils.hasText(properties.getBasename())) {
               //获取基础名
   			messageSource.setBasenames(StringUtils
   					.commaDelimitedListToStringArray(StringUtils.trimAllWhitespace(properties.getBasename())));
   		}
   		if (properties.getEncoding() != null) {
   			messageSource.setDefaultEncoding(properties.getEncoding().name());
   		}
   		messageSource.setFallbackToSystemLocale(properties.isFallbackToSystemLocale());
   		Duration cacheDuration = properties.getCacheDuration();
   		if (cacheDuration != null) {
   			messageSource.setCacheMillis(cacheDuration.toMillis());
   		}
   		messageSource.setAlwaysUseMessageFormat(properties.isAlwaysUseMessageFormat());
   		messageSource.setUseCodeAsDefaultMessage(properties.isUseCodeAsDefaultMessage());
   		return messageSource;
   	}
   ```

   这里会获取基础名，默认基础名是messages，即我们如果在resources下配置messages.properties的配置文件，国际化可以直接识别到，但是我们现在配置的基础名是login，国际化无法识别，因此我们需要在application.properties或者application.yml中配置基础名

   ```properties
   #配置国际化的基础名，如果有多个，需要配置多个
   spring:
     messages:
       basename: i18n.login
   ```

7. 使用国际化，前端工程如何使用国际化这里不再介绍，因为我们不用模板引擎，这里只介绍一个前台点击“中文”，“English”如何切换语言

   首先：点击“中文”发送send请求，携带信息：inter=zh_CN，点击“English”发送send请求，携带信息：inter=en_US

   后端接收到后进行处理如下：

   java\com\hui\internationalization\MyLocaleResolver.java

   ```java
   public class MyLocaleResolver implements LocaleResolver {
       @Override
       public Locale resolveLocale(HttpServletRequest request) {
           String inter = request.getParameter("inter");
           //默认使用操作系统使用的国际化语言
           Locale locale=Locale.getDefault();
           if(!StringUtils.isEmpty(inter)){
               String[] inters = inter.split("_");
               locale=new Locale(inters[0],inters[1]);
           }
           return locale;
       }
   
       @Override
       public void setLocale(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Locale locale) {
   
       }
   }
   ```

   然后把这个类加入到Spring容器中，关于国际化的LocaleResolver自动加载配置也在WebMvcAutoConfiguration中

   ```java
   @Configuration
   public class MyMvcConfig implements WebMvcConfigurer {
   
       @Bean
       public LocaleResolver localeResolver(){
           return new MyLocaleResolver();
       }
   }
   ```

#### 登录/拦截

登录这里不再介绍，介绍一下拦截，首先要写一个拦截器

com/hui/interceptor/LoginInterceptor.java

 ```java
public class LoginInterceptor implements HandlerInterceptor {

    //这个方法是在访问接口之前执行的，我们只需要在这里写验证登陆状态的业务逻辑，就可以在用户调用指定接口之前验证登陆状态了
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //每一个项目对于登陆的实现逻辑都有所区别，我这里使用最简单的Session提取User来验证登陆。有些从redis中取
        HttpSession session = request.getSession();
        //这里的User是登陆时放入session的
        User user = (User) session.getAttribute("username");
        //如果session中没有user，表示没登陆
        if (user == null){
            //这个方法返回false表示忽略当前请求，如果一个用户调用了需要登陆才能使用的接口，如果他没有登陆这里会直接忽略掉
            //当然你可以利用response给用户返回一些提示信息，告诉他没登陆
            return false;
        }else {
            return true;    //如果session里有user，表示该用户已经登陆，放行，用户即可继续调用自己需要的接口
        }

    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
 ```

然后使用这个拦截器

com/hui/config/MyMvcConfig.java

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    //添加拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //addPathPatterns("/**")表示对所有的请求进行拦截
        // excludePathPatterns("/login", "/register") 表示除了登陆与注册不拦截，因为登陆注册不需要登陆也可以访问
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/**").excludePathPatterns("/login", "/register");
    }
}
```

#### Restful

核心的一些注解：

@RestController

```java
@RestController
public class UserController {
```

@PathVariable：从请求中获取值

```java
@GetMapping("/user/{userId}")
public User findUser(@PathVariable Integer userId){
    return userService.findUser(userId);
}
```

| 接口URL    | HTTP方法 | 接口说明 |
| ---------- | -------- | -------- |
| /user      | POST     | 增       |
| /user/{id} | GET      | 查       |
| /user/{id} | DELETE   | 删       |
| /user/{id} | PUT      | 改       |