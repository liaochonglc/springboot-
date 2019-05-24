# SpringBoot

>  作用:整合了流行的第三方的框架,简化项目搭建操作,快速搭建spring应用



先描述一下一些区别于SpringMVC的注解

| 注解              | 意义                                                 |
| ----------------- | ---------------------------------------------------- |
| @RestContrller    | 视图解析器失效,不会跳转页面,return "abv",就是返回abv |
| @GetMapping       | @RequestMapping(method = RequestMethod.GET)的缩写    |
| @Configuration    | 标识配置类的注解                                     |
| @Bean             | 配置bean的注解,搭配@configuration使用                |
| @EnableScheduling | 开启之后默认添加到springBoot中,启动就会开始执行      |
| @Scheduled        | 定时器的注解                                         |
| @Mapper           | 单个mapper.xml的扫描(使用在mapper.xml中)             |
| @MapperScan       | 扫描mapper.xml的路径,用在SpringBoot启动类            |
|                   |                                                      |

## Bean加载

### Bean加载的几种方式

1. @Bean配置

   ```java
   @Bean
   public User getUser() {
   	return new User();
   }
   ```

2. 改造构造方法(要求不能使用定义无参构造方法)

   ```java
   @Component
   public class User {
       private ApplicationContext applicationContext;
   
       public User(ApplicationContext applicationContext) {
           this.applicationContext = applicationContext;
       }
   }
   ```

3. 实现ApplicationContextAware

   ```java
   public class User implements ApplicationContextAware {
   
   @Override
       public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
          this.applicationContext=applicationContext
       }
   }
   ```


### Bean加载的前后业务处理

>需要继承BeanPostProcessor

```java
@Component
public class EchoBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
//        System.out.println("----------------创建了这个" + bean.getClass()+"bean之前");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
//        System.out.println("----------------创建了这个" + bean.getClass()+"bean之后");

        return bean;
    }
}
```

### Bean加载的前后对象属性处理

>对一个bean初始化完成的时候进行处理, 所以叫bean工厂后置处理器

```java
/**
 * @Description: 对一个bean初始化完成的时候进行处理, 所以叫bean工厂后置处理器
 * 以下展示对user这个bean初始化完成之后赋值nickname为huwei
 * @Author: Awake-Hu
 * @Date: 2019/5/21
 */
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        BeanDefinition user = configurableListableBeanFactory.getBeanDefinition("user");
        MutablePropertyValues propertyValues = user.getPropertyValues();
        propertyValues.addPropertyValue("nickname", "huwei");
        user.setScope(BeanDefinition.SCOPE_SINGLETON);
    }
}
```





### @EnableAutoConfiguration

>相比@configuration注解,@EnableAutoConfiguration加载spring-boot-autoconfigure下的META-INF
>下的spring.factories中定义的bean类,用于自动装配外部jar包中的bean;@configuration更多用于加载本项目的bean
>
>注意spring.factories下的类都会默认注册一个无参的bean

自己定义一个外部项目，core-bean，依赖如下，

```
<artifactId>core-bean</artifactId>
<packaging>jar</packaging>

<dependencies>
   <dependency>
       <groupId>org.springframework</groupId>
       <artifactId>spring-context</artifactId>
       <version>4.3.9.RELEASE</version>
   </dependency>
</dependencies>
```

然后定义一个Cat类，

```
public class Cat {
}
package core.bean;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyConfig {

    @Bean
    public Cat cat(){
        return new Cat();
    }
}
```

我们知道这样就将Cat类装配到Spring容器了。

再定义一个springboot项目，加入core-bean依赖，依赖如下：

```
<artifactId>springboot-enableAutoConfiguration</artifactId>
<packaging>jar</packaging>

<dependencies>
     <dependency>
            <groupId>com.zhihao.miao</groupId>
            <artifactId>core-bean</artifactId>
            <version>1.0-SNAPSHOT</version>
      </dependency>
      <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
     </dependency>
</dependencies>
```

启动类启动：

```
@EnableAutoConfiguration
@ComponentScan
public class Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext context =SpringApplication.run(Application.class,args);
        Cat cat = context.getBean(Cat.class);
        System.out.println(cat);
    }
}
```



![img](https:////upload-images.jianshu.io/upload_images/5225109-b85d816485ae01cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



发现Cat类并没有纳入到springboot-enableAutoConfiguration项目中。

解决方案，
 在core-bean项目resource下新建文件夹META-INF，在文件夹下面新建spring.factories文件，文件中配置，key为自定配置类EnableAutoConfiguration的全路径，value是配置类的全路径

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=core.bean.MyConfig
```

启动springboot-enableAutoConfiguration项目，打印结果：



![img](https:////upload-images.jianshu.io/upload_images/5225109-9f72a8dc9f971035.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



### @ConditionalOnMissingBean

>表示当这个类型的bean时存在于spring容器,强行配置ConditionalOnBean注解的bean
>
>该类继承了condition注解的matcher方法,当方法返回值为true则装配,否则不装配

### @EnableConfigurationProperties

>注解配置类可以成为一个bean
>
>```java
>@Data
>//这里会提示在启动类需要@EnableConfigurationProperties注解使
>//下面的类可以成为bean注入到spring容器
>@ConfigurationProperties(prefix = "server")
>public class ServerPortConfig {
>    private String port;
>    private String name;
>}
>```



## 热部署

>  需要依赖和在`spring-boot-maven-plugin`配置`fork`

```xml
 
 <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
        
 <configuration>
<!--热部署需要的配置-->
<fork>true</fork>
</configuration>
```

![在这里插入图片描述](E:\typora\img\20181102175954987.png)




## 配置启动类

> 自动扫spring包

```
@ComponentScan(basePackages = {"com"})
```



## 配置过滤器

```java
  @Bean
    public FilterRegistrationBean<Filter> registFilter(){
        FilterRegistrationBean<Filter> filterFilterRegistrationBean = new FilterRegistrationBean<>();
        filterFilterRegistrationBean.setName("MyFlter");
        filterFilterRegistrationBean.setFilter(new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {

            }

            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                System.out.println("过滤前!");
                filterChain.doFilter(servletRequest, servletResponse);
                System.out.println("过滤之后!");
            }

            @Override
            public void destroy() {

            }
        });
        filterFilterRegistrationBean.addUrlPatterns("/*");
        return filterFilterRegistrationBean;


    }
```



自定义配置类

>  务必使用@configuration注解,可以把bean用@bean加载进该类,不使用@configuration就只能写进启动类

```java
@Configuration
public class Config {
	@bean
	public void test(){
	}
}
```

这里就不得不提到@configuration和@component的区别和联系了

>从定义来看， `@Configuration` 注解本质上还是 `@Component`，因此 `<context:component-scan/>` 或者 `@ComponentScan` 都能处理`@Configuration` 注解的类。 



## 自定义配置

语法:value(${config})

```properties
#applicatino.properties
server.port=8082
spring.mvc.view.prefix=/
spring.mvc.view.suffix=.jsp

my.name=wei
```

```java
public class Config {
    @Value("${my.name}")
    private String name;
    
    public void printName(){
        System.out.println(name);
       //输出:wei
    }
}
```

## yml配置

> 具有比properties更格式更美观的格式

允许

```properties
server:
  port: 8082

spring:
  mvc:
   view:
    prefix: /
    suffix: jsp
my:
  name: wei
```

### 1.配置多套yml配置文件

```properties
#将会匹配后缀为test的配置文件为启用状态
profiles:
    active: test
```



## 定时任务

### 1.传统的定时任务(java.util)

```java
@RequestMapping("/quart")
    public void quart() {
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("定时任务体" + new Date());
            }
        },0,2000);
        //后面两个参数的意思:多久之后执行,
        //比如String s = "2018-10-29" new simpleDateFormat().format("yyyy-MM-dd")
        //重复执行的间隔时间
    }

```

### 2.SpringBoot提供的定时任务(@Scheduled)

```java
 /*sprignBoot定时任务(2秒执行一次)*/
    @RequestMapping("/sbQuart")
    @Scheduled(fixedRate = 2000)
    public void taskTest() {
        System.out.println("定时任务已经执行!"+new Date());
    }

   //同时在类上方要加入@EnableScheduling注解,定时器才会执行
```



| 定时器配置值 |       意义       |
| :----------: | :--------------: |
|  fixedRate   | 多久之后开始执行 |
|     cron     | 执行的cron表达式 |

![在这里插入图片描述](E:\typora\img\2018110218012684.png)

[^cron表达式的几个例子]: 

[`cron表达式在线生成器`](http://cron.qqe2.com/)

![在这里插入图片描述](E:\typora\img\20181102180107652.png)

[^注:2018-10-1开始,每两周发布一次任务,直到12月]: 

​						

## 整合Mybatis

### 1.导入依赖

```xml
<--该依赖不是parent工程中的,所以需要制定version版本--> 
<dependency>
     <groupId>org.mybatis.spring.boot</groupId>
     <artifactId>mybatis-spring-boot-starter</artifactId>
     <version>1.3.0</version>
 </dependency>
```

该jar包具备了以下几个jar包(所以无需额外导入`mybatis_spring`,`mybatis`)

![在这里插入图片描述](E:\typora\img\20181102180158181.png)



### 2.配置mybatis 的 yml

 1. 配置数据源

    ```properties
     datasource:
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://172.16.1.255:3306/linayi?autoReconnect=true
        #这里千万不要写成 data-username: 
        username: dev_ess
        password: dev_ess
    ```

    2. 配置mapper映射和bo类映射以及dao扫包

    ```properties
    mybatis:
      mapper-locations: classpath:com/wei/mapper/*.xml
      type-aliases-package: classpath:com.wei.bo
    ```

    注意这里dao扫包有两种方式,类似于config扫包,

    - 从启动类加上`@MapperScan(basePackages = {"com.wei.dao"})`

    - 从每个userDao类上加上`@Mapper`

## log4j打印sql

>SpringBoot集成了log4j


![在这里插入图片描述](E:\typora\img\20181102180225251.png)


### 1.使用方法:

在`application.yml`中添加

```properties
mybatis.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```

格式化之后

```properties
mybatis:
 configuration:
  log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

## 异常类处理

### 过程

>对程序发生某一特定定义的异常的时候
>
>做出相应的反应(跳转页面,返回提示)
>
>关键优点:将 Controller 层的异常和数据校验的异常进行统一处理，减少模板代码，减少编码量，提升扩展性和可维护性。

##### 1.处理类上面加上`@ControllerAdvice`标志其是异常处理类

##### 2.加上`@ResponseBody`标志其返回json数据

##### 3.定义异常处理方法

```java
//ExceptionHandler标记受处理的异常类,如果不加参数表示拦截所有未标记的异常  
@ExceptionHandler(ArithmeticException.class)
    public UserBO doHandleError() {
        UserBO userBO = new UserBO();
        userBO.setName("请不要除1");
        return userBO;
    }
```

结果

![在这里插入图片描述](E:\typora\img\20181102180252591.png)

## 集成模板引擎thymeleaf

### 使用

##### 1.引入依赖

```xml
 <!--thymeleaf-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
            <version>2.0.5.RELEASE</version>
        </dependency>
    </dependencies>
```

##### 2.yml引入配置

```properties
 thymeleaf:
    cache: false
    prefix: classpath:/templates/
```

##### 3.添加接口

```java
@RequestMapping("/thymeleaf")
    public String thymeleaf(Model model) {
        model.addAttribute("name", "hu");
        model.addAttribute("age", "16");
        return "HelloThymeleaf";
    }
```



##### 4.填写页面

```html
<!DOCTYPE html SYSTEM "http://www.thymeleaf.org/dtd/xhtml1-strict-thymeleaf-4.dtd">

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">

<head>
    <title>Good Thymes Virtual Grocery</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <link rel="stylesheet" type="text/css" media="all"
          href="../../css/gtvg.css" th:href="@{/css/gtvg.css}" />
</head>

<body>

<p th:text="sadf">Welcome to our grocery store!</p>

</body>

</html>
```

![在这里插入图片描述](E:\typora\img\20181102180304841.png)

##### 5.thymelead字符拼接

- 用||包含

```html
th:text="|中文名:${name}|"
```

- 用''+拼接

```html
th:text="'中文名:'+${name}"
```

##### 6.thymelead条件判断

```html
<a th:if="${age>=18}">成年</a>
<a th:if="${age<18}">未成年</a>
<!--unless就是否的意思,代表相反-->
<a th:unless="${age>=18}">未成年</a>
```

##### 7.thymelead循环结构

```html
<a th:each="str,strState : ${eachStr}">
    <!--获取迭代的变量-->
	<a th:text="${str}"></a>
	<!--获取迭代的变量-->
    <a th:text="${strState.current }"></a>
	<!--获取迭代的后缀(从0开始)-->
	<a th:text="${strState.index}"></a>
	<!--获取迭代的后缀(从1开始)-->
    <a th:text="${strState.count}"></a>
	<!--获取迭代集合的长度-->
    <a th:text="${strState.size }"></a>
</a>
<a th:text="${#arrays.length(eachStr)}"></a>
```

##### 8.thymelead格式化

```html
<a th:text="${#dates.format(birthday,'yyyy-MM-dd')}">
```


## 部署

### 1.使用maven插件

### 2.启动

- 然后在war包所在的文件夹下输入 `java -jar demo-0.0.1-SNAPSHOT.war`

##### 3.访问

![在这里插入图片描述](E:\typora\img\20181102190731803.png)