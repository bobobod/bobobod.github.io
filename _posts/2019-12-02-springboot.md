---
layout: post
title: springboot学习笔记
date: 2019-12-02
tags: tools
---



## 一.配置文件

### 1.配置文件

---

#### YAML（YAML Ain’t Markup Language）

 	YAML A Markup Language：是一个标记语言

 	YAML isn’t Markup Language：不是一个标记语言；

#### 标记语言：

 	以前的配置文件；大多都使用的是 **xxxx.xml**文件；

 	YAML：**以数据为中心**，比json、xml等更适合做配置文件；

### 2.YAML语法

***

#### 1.基本语法

key:(空格)value  :表示一对键值对（空格必须有）

以**空格**的缩进来控制层级关系；只要左对齐的一列数据，都是同一层级

```yaml
server:
	port: 8081
	path: /hello
```

属性和值是大小写敏感的

#### 2.值的写法

##### 字面量：普通的值（数字，字符串，布尔）

​		k： v 字面直接来写；

​					字符串默认不用加上单引号或双引号

​					“”：双引号；不会转义字符串里面的特殊字符；特殊字符会中作为本身想表示的意思

​								name: "zhangsan \n lisi" 输出：zhangsan 换行 lisi

​					‘’：单引号；会转义字符中的特殊字符

​								name: "zhangsan \n lisi" 输出：zhangsan \n lisi

##### 对象，map（属性和值)（键值对）；

​	k: v :在下一行来写对象的属性和值的关系；注意缩进

​			对象还是k: v的方式

```yaml
friends:
	lastName: zhangsan
	age: 20
```

行内写法：

```yaml
friend: {lastName: zhangsan,age: 20}
```

##### 数组（List，Set）；

用- 值表示数组中的一个元素

```yaml
pets:
	- cat
	- dog
	- pig
```

行内写法：

```yaml
pets: [cat,dog,pig]
```

##### 具体样例

```yaml
#配置普通属性
name: zhangsan
#配置对象属性
#方式一
people:
  name: xiaoming
  age: 18
  addr: beijing
#方式二
#people: {name: xiaoming,age: 18,addr: beijing}
#配置数组，集合
city:
  - beijing
  - shanghai
  - tianjing
#方式二
#city: [beijing,shanghai,tianjing]
#配置数组，集合（对象）
person:
  - name: tom
    age: 19
    addr: beijing
  - name: lucy
    age: 20
    addr: beijing
#方式二
#person: [{name: tom,age: 19,addr: beijing}]
#map
map:
  key1: value1
  key2: value2
```

### 3.配置文件值注入

配置文件

```yaml
person:
	lastName: kiki
	#或last-name
	age: 18
	boss: false
	birth: 2019/12/12
	maps: {k1: v1,k2: v2}
	lists:
		- lisi
		- zhaoliu
	dog:
		name: 小狗
		age: 12
```

JavaBean

```java
/**
*将配置文件中的配置的每一个属性的值，映射到这个组件中
* @ConfigurationProperties:告诉springboot将本类的所有属性和配置文件一一绑定
* 	prefix="person":配置文件下的那个属性一一对应,默认是全局配置文件
*只有这个组件是容器中的组件，才能使用configurationProperties的功能
*/
@Conponent
@ConfigurationProperties(prefix="person")
public class Person{
    private String lastName;
    private Integer age;
    private Boolean boss;
    private Date birth;
    private Map<String,Object> maps;
    private List<Object> lists;
    private Dog dog;
}
```

我们可以导入配置文件处理器，以后编写配置就有提示了

```xml
<!--导入配置文件处理器，配置文件进行绑定就会有提示-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

**使用properties配置时可能中文会乱码，在setting->file encoding->选择Utf-8，并勾选转换ascill**

#### 1.@Value获取值和@ConfigurationProperties的比较

|                         | @configurationProperties             | @Value               |
| ----------------------- | :----------------------------------- | :------------------- |
| 功能上                  | 批量注入配置文件中的属性（javaBean） | 单个指定（单独使用） |
| 松散绑定                | 支持                                 | 不支持               |
| SpEL（#{SpEL}）         | 不支持                               | 支持                 |
| JSR303校验（@validated) | 支持                                 | 不支持               |
| 复杂类型封装            | 支持                                 | 不支持               |

#### 2.配置文件注入值数据校验

```java
@Conponent
@ConfigurationProperties(prefix="person")
@Validated
public class Person{
    @Email
    private String lastName;
    private Integer age;
    //@Value("#{11*2}") 支持
    private Boolean boss;
    private Date birth;
    //@Value("${person.maps}") 不支持
    private Map<String,Object> maps;
    private List<Object> lists;
    private Dog dog;
} 
```

#### 3.@PropertySource()和@ImportResource

**@PropertySource**:加载指定的配置文件；

```java
/**
*将配置文件中的配置的每一个属性的值，映射到这个组件中
* @ConfigurationProperties:告诉springboot将本类的所有属性和配置文件一一绑定
* 	prefix="person":配置文件下的那个属性一一对应,默认是全局配置文件
*只有这个组件是容器中的组件，才能使用configurationProperties的功能
*/
@PropertySource(value={"classpath:person.properties"})
@Conponent
@ConfigurationProperties(prefix="person")
public class Person{
    private String lastName;
    private Integer age;
    private Boolean boss;
    private Date birth;
    private Map<String,Object> maps;
    private List<Object> lists;
    private Dog dog;
}
```

#### **4.@ImportResource:**导入spring的配置文件，让配置文件里面的内容生效

spring Boot 里面没有spring的配置文件，我们自己自己编写的配置文件，也不能自动识别，想让spring的配置文件生效，加载进来：@ImportResource标注在一个配置类上

```java
@ImportResource(locations={"classpath:beans.xml"})
//导入spring的配置文件让其生效
```

springboot 推荐给容器中添加组件的方式，**推荐使用全注解**

1. 配置类=====spring配置文件
2. 使用@Bean给容器中添加组件

```java
/**
*@Configuration:指明当前类是一个配置类，用来替代之前的spring的配置文件
*在配置文件中用<bean></bean>标签添加组件
*/
@Configuration
public class MyAppConfig{
    //将方法的返回值添加到容器中；容器中这个组件默认的id是方法名
    @Bean
    public HelloService helloService(){
        System.out.println("@Bean给容器中添加主键了");
        return new HelloService();
    }
}
```

### 4.配置文件占位符

#### 1.随机数

```java
${random.value},${random.uuid},${random.int}
```

2.占位符获取之前配置的值，如果没有可以使用：指定默认值

```properties
person.last-name=zhangsan${random.uuid}
person.dog.name=${person.hello:hello}_dog
```

### 5.Profile

#### 1.多Profile文件

在主配置文件编写的时候，文件名可以是application-{profile}.properties/yml

默认使用application.properties

#### 2.yml支持多文档块方式

```yml
server:
	port: 8081
spring:
	profiles:
		active: dev #指定属性那个配置
---
server:
	port: 8083
spring:
	profiles: dev
---
server:
	port: 8084
spring:
	profiles: prod
```



#### 3.激活指定profile

1. 在配置文件中指定 spring.profiles.active=dev

2. 命令行：（program arguments)

   --spring.profiles.active=dev

3. 虚拟机参数(vm option)

   -Dspring.profiles.active=dev

### 6.配置文件的加载位置

​	springboot启动会扫描以下位置的application.properties/yml文件作为springboot的默认的配置文件

1. file：./config/ 注：当前项目的根目录下
2. file: ./
3. classpath:/config/
4. classpath:/

**以上是按照优先级从高到低的顺序，所有位置的文件都会被加载，高优先级配置的内容会覆盖低优先级配置内容**

**我们还可以通过spring.config.location来改变配置文件的位置**

项目打包好以后，我们可以使用命令行参数的形式来指定配置文件的新位置;指定配置文件和默认加载的这些配置文件会共同起作用，**形成互补配置**

### 7.自动配置原理

#### 自动配置原理：	

1> springboot启动的时候加载了注配置类，开启了自动配置功能@EnableAutoConfiguration

2> @enableAutoConfiguration的作用

​	**利用插件selectImports()方法导入组件，将类路径下META-INF/spring.factories里面配置所有EnableAutoConfiguration的值导入容器中**

## 二.日志系统

---

### 1.日志框架

市面上的日志框架

JUL,JCL,Jboss-logging,logback,log4j,log4j2,slf4j

| 日志门面（日志抽象层）                                       | 日志实现                                          |
| ------------------------------------------------------------ | ------------------------------------------------- |
| ~~JCL(Jakarta Commons Logging)~~ SLF4J(simple logging facade for java) ~~jboss-logging~~ | log4j    ,jul(java.utils.logging),log4j2 ,logback |

左边选一个门面，右边来选一个实现

日志门面：SLF4J

日志实现：Logback

springboot：底层是spring 框架，spring框架默认使用jcl；

**springboot选用slf4j和logback**

### 2.slf4j的使用

#### 1.如何在系统中使用slf4j

调用日志抽象层的方法

给系统导入slf4j的jar和logback的实现jar

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloWorld {
  public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger(HelloWorld.class);
    logger.info("Hello World");
  }

```

![](http://www.slf4j.org/images/concrete-bindings.png)

**每个日志的实现框架都有自己的配置**，**配置文件还是做日志实现的配置文件**

#### 2.遗留问题

统一日志记录，即使是别的框架和我一样统一使用slf4j输出？

1. 将系统中其他日志框架先排除除去
2. 用中间包来替换原有的日志框架（springboot也是这样操作的）
3. 我们导入slf4j其他的实现
4. 如果我们要引用其他框架？**一定要把这个框架默认的日志框架排除掉**

![](http://www.slf4j.org/images/legacy.png)

### 3.日志的使用

#### 1.默认配置

springboot默认将我们配置好了日志

```java
//记录器
Logger logger = LoggerFactory.getLogger(getClass());
//日志级别：
//由低到高 trace<debug<info<warn<error
//可以调整输出的日志级别,输入高级别
logger.trace("这是trace日志");
logger.debug("这是debug日志");
//springboot默认是info级别
logger.info("这是info日志");
logger.warn("这是warn日志");
logger.error("这是error日志");
```

```properties
#指定日志级别和包目录
logging.level.com.cczu=trace
#不指定路径在当前项目下生成springboot.log
#可以指定完整路径,logging.file和logging.path二选一
logging.file=G:/springboot.log
logging.path=/spring/log
#在控制台输出日志的格式
logging.pattern.console=
#指定文件中日志文件的格式
# %-5level :级别从左显示5个字符宽度
logging.pattern.file=%d{yyyy-MM-dd} [%thread] %-5level %logger{50} - %msg%n
```

## 三.Web开发

---

### 1.使用springboot

1. 创建springboot应用，选中需要的模块
2. springboot已经默认将这些场景配置好了，只需指定少量的配置就可以运行起来？
3. 自己编写业务代码

自动配置原理？

这个场景spring帮我们配置了什么？能不能修改？能修改哪些配置？能不能拓展？

```properties
****AutoConfiguration:帮我们给容器自动配置组件
****Properties:配置类来封装配置文件的内容
```

### 2.springboot对静态资源的映射规则

```java
@ConfigurationProperties(
    prefix = "spring.resources",
    ignoreUnknownFields = false
)
public class ResourceProperties {
 
//可以设置和静态相关的参数，缓存时间
```



```java
//静态资源映射    
public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                logger.debug("Default resource handling disabled");
            } else {
                Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
                CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
                if (!registry.hasMappingForPattern("/webjars/**")) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

                String staticPathPattern = this.mvcProperties.getStaticPathPattern();
                if (!registry.hasMappingForPattern(staticPathPattern)) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations())).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
                }

            }
        }

```

1. 所有/webjars/**,都去"classpath:/META-INF/resources/webjars/"找资源

   - webjars:以jar包的方式引入静态资源

   [webjars]: https://www.webjars.org/

   ![](/images/posts/webjars.png)

访问:localhost:8080/webjars/jquery/3.4.1/jquery.js

引入依赖

```xml
    <dependency>
            <groupId>org.webjars</groupId>
            <artifactId>jquery</artifactId>
            <version>3.4.1</version>
        </dependency>
```

2. /**访问当前项目的任何资源（静态资源文件夹）

```properties
 "classpath:/META-INF/resources/",
 "classpath:/resources/",
 "classpath:/static/", 
 "classpath:/public/"
# / 项目的根路径
```

访问：localhsot:8080/js/jquery.js

3. 欢迎页设置：静态资源下的index.html,规则是/**

```java
//欢迎页
       @Bean
        public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
            WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(new TemplateAvailabilityProviders(applicationContext), applicationContext, this.getWelcomePage(), this.mvcProperties.getStaticPathPattern());
            welcomePageHandlerMapping.setInterceptors(this.getInterceptors(mvcConversionService, mvcResourceUrlProvider));
            return welcomePageHandlerMapping;
        }


      private Optional<Resource> getWelcomePage() {
            String[] locations = WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations());
            return Arrays.stream(locations).map(this::getIndexHtml).filter(this::isReadable).findFirst();
        }

        private Resource getIndexHtml(String location) {
            //看这里
            return this.resourceLoader.getResource(location + "index.html");
        }
//映射规则
   this.staticPathPattern = "/**";
```

访问：localhost:8080/         会去静态资源下的index.html找

4. 所有***/favicon.ico都是从静态资源文件找
5. 也可以自行配置

```properties
spring.resources.static-location=classpath:/hello/
```



### 3.模板引擎

springboot内嵌的tomcat是不支持jsp的

jsp，velocity，freemarker,**thymeleaf**

springBoot推荐使用Thymeleaf

#### 1.引入thymeleaf

```xml
<thymeleaf.version>3.0.11.RELEASE</thymeleaf.version>
<!--thymeleaf3主程序 layout2主程序 -->
<thymeleaf-layout-dialect.version>2.4.1</thymeleaf-layout-dialect.version>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

#### 2.Thymeleaf使用和语法

```java
@ConfigurationProperties(
    prefix = "spring.thymeleaf"
)
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    private boolean checkTemplate = true;
    private boolean checkTemplateLocation = true;
    private String prefix = "classpath:/templates/";
    private String suffix = ".html";
    private String mode = "HTML";
    //只要我们把html页面放在classpath：/templates/,thymeleaf会自动渲染
    ...

```

使用：

1.导入thymeleaf的名称空间，会有提示

```html
<html xmlns:th="http://www.thymeleaf.org">
```

2.使用thymeleaf语法

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
hello world
<!--    th:text会替换文本值-->
<div th:text="${hello}">这是欢迎</div>
</body>
</html>
```

3.语法规则

1. ##### th:text 修改当前元素里面的文本内容

   th：任意html属性；来替换原生属性的值  th:id

   ![](/images/posts/thymeleaf.png)

2. ##### 表达式

```properties
Simple expressions:(表达式语法)
    Variable Expressions: ${...} 获取变量的值
    		1. 获取对象的属性，调用方法
    		2.使用内置的基本对象：
    			#ctx : the context object.
                #vars: the context variables.
                #locale : the context locale.
                #request : (only in Web Contexts) the HttpServletRequest object.
                #response : (only in Web Contexts) the HttpServletResponse object.
                #session : (only in Web Contexts) the HttpSession object.
                #servletContext : (only in Web Contexts) the ServletContext object.
              3.内置的一些工具对象
              	#execInfo : information about the template being processed.
                #messages : methods for obtaining externalized messages inside variables expressions, in the same way as they
                would be obtained using #{...} syntax.
                #uris : methods for escaping parts of URLs/URIs
                Page 20 of 106#conversions : methods for executing the configured conversion service (if any).
                #dates : methods for java.util.Date objects: formatting, component extraction, etc.
                #calendars : analogous to #dates , but for java.util.Calendar objects.
                #numbers : methods for formatting numeric objects.
                #strings : methods for String objects: contains, startsWith, prepending/appending, etc.
                #objects : methods for objects in general.
                #bools : methods for boolean evaluation.
                #arrays : methods for arrays.
                #lists : methods for lists.
                #sets : methods for sets.
                #maps : methods for maps.
                #aggregates : methods for creating aggregates on arrays or collections.
                #ids : methods for dealing with id attributes that might be repeated (for example, as a result of an iteration).
    Selection Variable Expressions: *{...}   选择表达式 和${}是一样的
    	补充:配合th：object使用
    	#<div th:object="${session.user}">
        #<p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
        #<p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
        #<p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
        #</div>
    Message Expressions: #{...}  取国际化内容
    Link URL Expressions: @{...} 定义url链接
    	@{/order/process(execId=${execId},execType='FAST')}
    Fragment Expressions: ~{...} 片段引用表达式
    	<div th:insert="~{commons :: main}">...</div>
Literals 字面量
    Text literals: 'one text' , 'Another one!' ,...
    Number literals: 0 , 34 , 3.0 , 12.3 ,...
    Boolean literals: true , false
    Null literal: null
    Literal tokens: one , sometext , main ,...
Text operations:（文本操作）
    String concatenation: +
    Literal substitutions: |The name is ${name}|
Arithmetic operations:数学运算
    Binary operators: + , - , * , / , %
    Minus sign (unary operator): -
Boolean operations:布尔运算
    Binary operators: and , or
    Boolean negation (unary operator): ! , not
Comparisons and equality:比较运算
    Comparators: > , < , >= , <= ( gt , lt , ge , le )
    Equality operators: == , != ( eq , ne )
Conditional operators:条件运算
    If-then: (if) ? (then)
    If-then-else: (if) ? (then) : (else)
    Default: (value) ?: (defaultvalue)
Special tokens:特殊运算

```

### 4.SpringMvc自动配置

#### 1.springmvc自动配置Mvc

springboot自动配置好了springmvc

以下是springboot对springmvc的默认

1. 自动配置了viewResolver（视图解析器：根据方法的返回值得到视图对象,视图对象决定如何渲染（转发或渲染？））和BeanNameViewResolver beans

   - ContentNegotiatingViewResolver：组合所有视图解析器；
   - 如何定值：我们可以自己给容器添加一个视图解析器，自动组合进来

2. 支持静态资源文件夹路径和webjars

3. 静态首页访问

4. 自定义favicon.ico

5. 自动注册了Converter,GenericController,Formatter beans

   - Converter:转换器；类型转换使用converter
   - Formatter 格式化；2017-12-17===Date；
   - 自己添加的格式化器转换器，我们只需要放在容器中即可

6. 支持HttpMessageConverter功能

   - httpMessageConverter:springvc用来转换User-json

   - httpMessageConverter是从容器中确定的；获取所有httpMessageConverter

   - 自己添加的HttpMessageConverter器转换器，我们只需要放在容器（@Bean，@Conponent)中即可
   
7. 自动注册MessageCodeResolver:定义错误代码规则
   
8. 自动使用ConfigurableWebBindingInitializer bean

   - 我们可以配置一个以上的类来替换**默认**的；（添加到容器）
   - 初始化WebDataBinder；请求数据===JavaBean

#### 2.拓展springmvc

```xml
<mvc:view-controller path="/hello" view-name="success" />
<mvc:interceptors>
	<mvc:interceptor>
    	<mvc:mapping path="/hello" />
        <bean></bean>
    </mvc:interceptor>
</mvc:interceptors>
```

**编写一个配置类（@Configuration),是WebMvcConfigurationAdapter类型(该类已废弃），可以实现WebMvcConfiguration接口；不能标注@EnableWebMvc**。即保留了默认的的自动配置，也能用我们拓展的配置

```java
@Configuration
public class MvcConfig  implements WebMvcConfigurer {
	@Override
	public void addViewControllers(ViewControllerRegistry registry) {
		registry.addViewController("/atguigu").setViewName("success");
	}

```

**原理：**

1. WebMvcAutoConfiguration是springmvc的自动配置类
2. 在做其他自动配置时会自动导入EnableWebMvcConfiguration类

```java
@Configuration(
    proxyBeanMethods = false
)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware 

//从容器中获取所有的webmvcconfigurer
        @Autowired(
    required = false
)
        public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
            //一个参考实现：将所有webmvcconfigurer相关配置都来一起调用
          //  public void addViewControllers(ViewControllerRegistry registry) {
            //    Iterator var2 = this.delegates.iterator();

              //  while(var2.hasNext()) {
                //    WebMvcConfigurer delegate = (WebMvcConfigurer)var2.next();
                  //  delegate.addViewControllers(registry);
               // }

            //}
        }

    }
```

3. 容器中所有的WebMvcConfigurer都会一起起作用
4. 我们的配置类也会被调用，springmvc的自动配置和我们的拓展配置都会起作用

#### 3.全面接管springmvc

​	springBoot对springmvc的自动配置不需要了，所有都要自己配置

​	**只需要在配置类中加入@EnableWebMvc**

```java
@EnableWebMvc
@Configuration
public class MvcConfig  implements WebMvcConfigurer {
	@Override
	public void addViewControllers(ViewControllerRegistry registry) {
		registry.addViewController("/atguigu").setViewName("success");
	}
}
```

**原理**：

为什么@EnableWebMvc自动配置会失效

1. @EnableWebMvc的核心

```java
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
```

2. 

```java
@Configuration(
    proxyBeanMethods = false
)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport{}
```

3. 查看webmvcautoconfiguration类

```java

@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
//容器中没有这个组件的时候，这个配置类才会生效
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {}
```

4. @EnableWebMvc将WebMvcConfigurationSupport导入进来了
5. 导入的WebMvcConfigurationSupport只是springmvc最基本的功能

### 5.如何修改springboot默认配置

模式：

1. springboot在自动配置很多组件的时候，先看用户有没有自己配置了（@Bean，@Component)如果有就用用户配置的，否则用默认的；如果有些组件可以有多个（ViewResolver）将用户配置和自己默认的组合进来
2. 在springboot中会有非常多的***Configurer类，需多留心
3. 在springboot中会有很多的xxxCustomizer来帮我们定制

### 6.Restful CRUD

#### 1.默认访问首页，默认访问静态资源文件夹，可以通过配置解决

```java
	//所有WebMvcConfigurer组件都会一起起作用
	@Bean
	//将组件注入容器
	public WebMvcConfigurer webMvcConfigurer(){
		WebMvcConfigurer adapter =new WebMvcConfigurer() {
			@Override
			public void addViewControllers(ViewControllerRegistry registry) {
				registry.addViewController("/").setViewName("login.html");
				registry.addViewController("/login.html").setViewName("login.html");
			}
		};
		return  adapter;

```



####  2.国际化

- **编写国际化配置文件**

- 使用ResourceBundleMessageSource管理国际化资源文件
   - 在页面使用fmt:message去除国际化内容

   步骤：
   
   1. 编写配置文件（login_zh_CN.properties)

```properties
login.btn=登录~
login.password=密码~`
login.remember=记住我~
login.tip=请登录~
login.username=用户名~
```

2. springboot自动配置好了管理国际化文件的组件

```java
@EnableConfigurationProperties
public class MessageSourceAutoConfiguration {
    private static final Resource[] NO_RESOURCES = new Resource[0];

    public MessageSourceAutoConfiguration() {
    }

    @Bean
    @ConfigurationProperties(
        prefix = "spring.messages"
    @Bean
    public MessageSource messageSource(MessageSourceProperties properties) {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        if (StringUtils.hasText(properties.getBasename())) {
           
 	//设置国际化资源文件的基础名（去掉国家语言和国家名的）
            messageSource.setBasenames(StringUtils.commaDelimitedListToStringArray(StringUtils.trimAllWhitespace(properties.getBasename())));
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
}
public class MessageSourceProperties {
 	//我们可以把配置文件直接放在类路径下叫message.properties
    private String basename = "messages";
}
```

设置配置

```properties
spring.messages.basename=i18n.login
#包名.login，默认是classpath
#注意中文可能乱码 需设置file encoding
```

```html
<!doctype html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta name="description" content="">
    <meta name="author" content="Mark Otto, Jacob Thornton, and Bootstrap contributors">
    <meta name="generator" content="Jekyll v3.8.5">
    <title>Floating labels example · Bootstrap</title>

    <link rel="canonical" href="https://getbootstrap.com/docs/4.3/examples/floating-labels/">

    <!-- Bootstrap core CSS -->
    <link th:href="@{/webjars/bootstrap/4.3.1/css/bootstrap.min.css}" rel="stylesheet"
          integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">


    <style>
        .bd-placeholder-img {
            font-size: 1.125rem;
            text-anchor: middle;
            -webkit-user-select: none;
            -moz-user-select: none;
            -ms-user-select: none;
            user-select: none;
        }

        @media (min-width: 768px) {
            .bd-placeholder-img-lg {
                font-size: 3.5rem;
            }
        }
    </style>
    <!-- Custom styles for this template -->
    <link href="https://getbootstrap.com/docs/4.3/examples/floating-labels/floating-labels.css" rel="stylesheet">
</head>
<body>
<form class="form-signin">
    <div class="text-center mb-4">
        <img class="mb-4" src="https://getbootstrap.com/docs/4.3/assets/brand/bootstrap-solid.svg" alt="" width="72"
             height="72">
        <h1 class="h3 mb-3 font-weight-normal" th:text="#{login.tip}">Floating labels</h1>
        <p>Build form controls with floating labels via the <code>:placeholder-shown</code> pseudo-element. <a
                href="https://caniuse.com/#feat=css-placeholder-shown">Works in latest Chrome, Safari, and Firefox.</a>
        </p>
    </div>

    <div class="form-label-group">
        <input type="email" id="username" class="form-control"  placeholder="Username" required autofocus>
        <label for="username" th:text="#{login.username}">username</label>
    </div>

    <div class="form-label-group">
        <input type="password" id="inputPassword" class="form-control" placeholder="Password" required>
        <label for="inputPassword" th:text="#{login.password}">Password</label>
    </div>

    <div class="checkbox mb-3">
        <label>
            <input type="checkbox" value="remember-me" /> [[#{login.remember}]]
        </label>
    </div>
    <button class="btn btn-lg btn-primary btn-block" type="submit" th:text="#{login.btn}">Sign in</button>
    <p class="mt-5 mb-3 text-muted text-center">&copy; 2017-2019</p>
    <a class="btn btn-sm">中文</a>
    <a class="btn btn-sm">English</a>
</form>
</body>
</html>

```

**通过浏览器切换语言达到效果**

原理：

- 国际化Locale（区域信息对象）；LocaleResovler(获取区域信息)

```java
@Bean
@ConditionalOnMissingBean
@ConditionalOnProperty(
    prefix = "spring.mvc",
    name = {"locale"}
)
public LocaleResolver localeResolver() {
    if (this.mvcProperties.getLocaleResolver() == org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.mvcProperties.getLocale());
    } else {
        AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
        localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
        return localeResolver;
    }
}
//默认是根据请求头带来的区域信息获取locale区域信息
```

通过链接修改locale信息

1. 配置localResolver解析器

```java
public class MyLocaleResolver implements LocaleResolver {
    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        String l = request.getParameter("l");
        Locale locale = Locale.getDefault();
        if(!StringUtils.isEmpty(l)){
            String[] split = l.split("_");
            locale = new Locale(split[0],split[1]);
        }
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Locale locale) {

    }
}

```

2. 配置bean

```java
@Bean
public MyLocaleResolver myLocaleResolver(){
    return new MyLocaleResolver();
}
```

#### 3.用户登录

开发期间模板引擎页面修改以后，要实时生效

1. 禁用模板引擎的缓存
2. 页面修改后重新build一下

```properties
#禁用模板引擎的缓存
spring.thymeleaf.cache=false
#修改页面后可能还是不生效，可以重新build一下 ctrl+f9
```

登录失败显示错误信息

```html
 <p style="color: red;" th:text="${msg}" th:if="${not #strings.isEmpty(msg)}">
 </p>
```

#### 4.请求拦截

```java

public class LoginIntercepter implements HandlerInterceptor {
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		String username = (String) request.getSession().getAttribute("username");
		if(!StringUtils.isEmpty(username)){
			return true;
		}else{
			request.getRequestDispatcher("/login.html").forward(request,response);
			return false;
		}
	}
}
//添加拦截器
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LoginIntercepter()).addPathPatterns("/**").excludePathPatterns("/","/login.html","/user/login");

```

#### 5.curd员工列表

实验要求：

1. restfulCRUD,满足Rest风格
   - uri： /资源名称/资源标识 http请求区分对资源crud操作

|      | 普通crud（uri来区别） | RestfulCRUD       |
| ---- | --------------------- | ----------------- |
| 查询 | getEmp                | emp---Get         |
| 添加 | addEmp?xxx            | emp--Post         |
| 修改 | updateEmp?id=xxx      | emp/{id} -- Put   |
| 删除 | deleteEmp?id=1        | emp/{id} --delete |

2. 使用架构

|                      | 请求uri  | 请求方式 |
| -------------------- | -------- | -------- |
| 查询所有员工         | emps     | GET      |
| 查询某个员工         | emp/{id} | GET      |
| 来到添加页面         | emp      | GET      |
| 添加员工             | emp      | POST     |
| 来到修改页面（回显） | emp/{id} | GET      |
| 修改员工             | emp/{id} | POST     |
| 删除员工             | emp/{id} | DELETE   |

3. 员工列表

thymeleaf公共页面元素抽取

```html
1.抽取公共片段
<div th:fragment="copy">
    hello world
</div>
2.引入片段
<div th:insert="~(footer :: copy)">
</div>
~(templatename::selector):模板名：：选择器
~(templatename::fragmentname):模板名：：片段名
在dashboard中抽取片段，list中使用片段
3.默认效果
insert的功能片段在div标签中
如果使用th:insert等属性引入，可以不写~{}
行内元素可以加上[[~{}]];[(~{})]
```

##### 三中引入功能片段的th属性

**th:insert** ：将公共片段完整的插入到div中

**th:replace** ：将声明引入的元素替换为公共元素

**th:include** ：将片段内容包含进div中

```html
<footer th:fragment="copy">
&copy; 2011 The Good Thymes Virtual Grocery
</footer>
插入片段
<body>
...
<div th:insert="footer :: copy"></div>
<div th:replace="footer :: copy"></div>
<div th:include="footer :: copy"></div>
</body>
结果是：
<body>
...
<div>
    <footer>
        &copy; 2011 The Good Thymes Virtual Grocery
    </footer>
</div>
<footer>
&copy; 2011 The Good Thymes Virtual Grocery
</footer>
<div>
&copy; 2011 The Good Thymes Virtual Grocery
</div>
</body>
```

引入片段的时候传入参数

```html
<div th:replace="~{common/bar::sidebar(activeUri='emps.html')}"></div>
```

#### 6.员工添加

**员工添加最容易引起格式不对，Bad Request 400**

2019-03-20 2019/01/20 2019.03.20

默认格式化是 / 转换为date

```properties
#日期格式化
spring.mvc.date-format=yyyy-MM-dd
```

#### 7.员工修改

##### 1.发送put请求修改员工数据

1. 在springmvc中配置hiddenhttpmethodfilter（springboot自动配置好了）
2. 页面创建一个post表单
3. 创建一个input项，name="_method",值就是我们请求的方式 value="put"

#### 2.springboot默认关闭了hiddenhttpmethodfilter

```properties
# 开启mvc的HiddenHttpMethodFilter，以便可以表单可以发送PUT、DELETE等请求
spring.mvc.hiddenmethod.filter.enabled=true
```



### 7.错误处理机制

#### 1.springboot默认的处理机制

默认效果

1. 浏览器，返回一个默认的错误页面,浏览器发送的请求头里面有个 Accept:text/html
2. 如果是其他客户端默认响应一个json数据,postman中优先接收/*

原理：

​	可以参照ErrorMvcAutoConfiguration:错误处理的自动配置

​	给容器中添加了以下组件

1. DefaultErrorAttributes

   ```text
   帮我们在页面共享信息
   ```

   

2. BasicErrorController

   ```java
   默认处理/error请求
   @Controller
   @RequestMapping({"${server.error.path:${error.path:/error}}"})
   public class BasicErrorController extends AbstractErrorController {
     @RequestMapping(
           produces = {"text/html"} //产生html类型的数据
       )
       public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
           HttpStatus status = this.getStatus(request);
           Map<String, Object> model = Collections.unmodifiableMap(this.getErrorAttributes(request, this.isIncludeStackTrace(request, MediaType.TEXT_HTML)));
           response.setStatus(status.value());
           ModelAndView modelAndView = this.resolveErrorView(request, response, status, model);
           return modelAndView != null ? modelAndView : new ModelAndView("error", model);
       }
   
       @RequestMapping //参数json
       public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
           HttpStatus status = this.getStatus(request);
           if (status == HttpStatus.NO_CONTENT) {
               return new ResponseEntity(status);
           } else {
               Map<String, Object> body = this.getErrorAttributes(request, this.isIncludeStackTrace(request, MediaType.ALL));
               return new ResponseEntity(body, status);
           }
       }
   ```

   

3. ErrorPageCustomizer

   ```java
   @Value("${error.path:/error}")
   private String path = "/error";系统出现错误以后来到error请求处理
   ```

   

4. DefaultErrorViewResolver

   ```java
       private ModelAndView resolve(String viewName, Map<String, Object> model) {
           //默认springboot可以取找一个页面 error/404
           String errorViewName = "error/" + viewName;
           //模板引擎解析这个页面地址作为模板引擎解析
           TemplateAvailabilityProvider provider = this.templateAvailabilityProviders.getProvider(errorViewName, this.applicationContext);
           //模板引擎可用的情况下返回errorviewName指定的视图地址
           return provider != null ? new ModelAndView(errorViewName, model) : this.resolveResource(errorViewName, model);
       }
       private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
           //模板引擎不可用，就在静态资源下找errorViewName对应的页面
           String[] var3 = this.resourceProperties.getStaticLocations();
           int var4 = var3.length;
   
           for(int var5 = 0; var5 < var4; ++var5) {
               String location = var3[var5];
   
               try {
                   Resource resource = this.applicationContext.getResource(location);
                   resource = resource.createRelative(viewName + ".html");
                   if (resource.exists()) {
                       return new ModelAndView(new DefaultErrorViewResolver.HtmlResourceView(resource), model);
                   }
               } catch (Exception var8) {
               }
           }
   
           return null;
       }
   ```

步骤：

​	一旦系统出现4xx或者5xx之类时，ErrorPageCustomizer就会生效（定制错误的响应规则），就会来到/error请求，会被baseErrorController处理,响应页面，去哪个页面又DefaultErrorViewController处理

#### 2.定制错误响应：

1. 如何定制错误界面

   1. 有模板引擎下；error/状态码；将错误命名为 错误代码.html 放在模板引擎下的error下。我们可以使用4xx和5xx作为错误页面的文件，优先选择精确的状态码.html

      页面能获取的信息：

      ​	timestamp：时间戳

      ​	status:状态码

      ​	error:错误提示

      ​	exception：异常对象

      ​	message：异常消息

      ​	errors:jsr303数据校验

   2. 没有模板引擎（模板引擎找不到），默认在静态资源文件夹下找，无法获取动态数据

   3. 以上都没有，来到springboot默认页面

2. 如何定制错误的json数据

   1. 自定义异常处理&返回定制json数据

```java
@ControllerAdvice
public class MyExceptionHandler {
    //浏览器客户端放回的都是json
	@ResponseBody
	@ExceptionHandler(UserNotExistException.class)
	public Map<String,Object> handlerException(Exception e){
		Map<String,Object> map=new HashMap<>();
		map.put("code","user not exist");
		map.put("message",e.getMessage());
		return map;
	}
}
//没有自适应效果
```

2. 转发到/error

```java
@ExceptionHandler(UserNotExistException.class)
public String handlerException(Exception e,HttpServletRequest request){
    Map<String,Object> map=new HashMap<>();
    //传入我的自己的错误状态码  4xx 5xx 否则不会来到自己定制的页面
    request.setAttribute("javax.servlet.error.status_code",400);
    map.put("code","user not exist");
    map.put("message",e.getMessage());
    //转发到/error
    return "forward:/error";
}
//不能携带自己定制的数据
```

3. 将我们定制的数据携带出去

   出现错误，会来到/error请求，会被basicErrorController处理，响应出去的数据是由getErrorAttriAttributes得到的（是 AbstractErrorController（ErrorController)的子类规定的子类）

   1. 完全来编写一个ErrorController的实现类（或者编写AbstractErrorController），放在容器中。

   2. 页面上能用的数据，或者是json返回能用的数据都是通过errorAttributes.getErrorAttributes得到的；

      容器中DefaultErrorAttributes默认添加。

      **自定义ErrorAttributes**

   ```java
   //给容器中添加组件
   @Component
   public class MyErrorAttributes extends DefaultErrorAttributes {
   	@Override
   	public Map<String, Object> getErrorAttributes(WebRequest webRequest, boolean includeStackTrace) {
   		Map<String, Object> map = super.getErrorAttributes(webRequest, includeStackTrace);
   		map.put("company","alibaba");
   		// 0 表示request，1表示session
   		Map<String,Object> ext= (Map<String, Object>) webRequest.getAttribute("ext", 0);
   		map.put("ext",ext);
   		return map;
   	}
   }
   
   @ControllerAdvice
   public class MyExceptionHandler {
   	@ResponseBody
   	@ExceptionHandler(UserNotExistException.class)
   	public Map<String,Object> handlerException(Exception e, HttpServletRequest request){
   		Map<String,Object> map=new HashMap<>();
   		//传入我的自己的错误状态码  4xx 5xx 否则不会来到自己定制的页面
   		request.setAttribute("javax.servlet.error.status_code",400);
   		map.put("code","user not exist");
   		map.put("message","用户出错了");
   		request.setAttribute("ext",map);
   		return map;
   	}
   }
   ```

   

### 8.配置嵌入式Servlet容器

####    1.SpringBoot默认使用的是嵌入式的Servlet容器（Tomcat）

问题？

1. 如何定制和修改Servlet容器的相关配置

   1. 修改和server有关的配置(ServerProperties )

      ```properties
      server.post=8081
      server.context-path=/crud
      
      server.tomcat.uri-encoding=UTF-8
      #通用的servlet容器设置
      server.xxx
      #Tomcat的设置
      server.tomcat.xxx
      ```

   2. ~~编写一个EmbeddedSevletContainerCustomizer:嵌入式的Servlet容器的定制器来修改servlet~~

   

#### 2.注册servlet三大组件

由于springboot默认是以jar包的方式启动嵌入式的servlet容器来启动web应用，没有web.xml文件

注册三大组件用以下方式

**servletRegisterationBean**   继承HttpServlet

```java
@Bean
public ServletRegistrationBean servletRegistrationBean(){
    ServletRegistrationBean myServletServletRegistrationBean = new ServletRegistrationBean(new MyServlet(), "/myservlet");
    return myServletServletRegistrationBean;
}
```

**FilterRegisteratationBean**  实现Filter

```java
@Bean
public FilterRegistrationBean filterRegistrationBean(){
    FilterRegistrationBean<Filter> filterFilterRegistrationBean = new FilterRegistrationBean<>();
    filterFilterRegistrationBean.setFilter(new MyFilter());
    filterFilterRegistrationBean.setUrlPatterns(Arrays.asList("/hello","/myservlet"));
    return filterFilterRegistrationBean;
}
```



**ServletListenerRegistrationBean**  实现ServletContextListener

```java
@Bean
public ServletListenerRegistrationBean servletListenerRegistrationBean(){
    ServletListenerRegistrationBean<MyListener> myListenerServletListenerRegistrationBean = new ServletListenerRegistrationBean<>(new MyListener());
    return myListenerServletListenerRegistrationBean;

}
```

**SpringBoot帮我们自动SpringMvc的时候，自动注册SpringMvc的前端控制器；DispatcherServlet;默认拦截方式是/ ,不拦截jsp，拦截静态资源，可以通过server.servletPath来修改前端控制器默认拦截请求的路径**

#### 3.替换其他容器

1.springboot能不能支持其他的servlet容器

可以，默认是tomcat，可以通过排除tomcat的依赖，加入其他容器的依赖

Jetty(长连接)

Undertow(不支持Jsp)

#### 4.嵌入式Servlet自动配置的原理

步骤：

1. springboot根据导入的依赖情况，给容器中导入相应的EmbeddedServletContaionerFactory
2. 容器中某个组件要创建对象就会惊动后置处理器
3. 只要是嵌入式的srevlet容器工厂，后置处理器工作
4. 从容器中获取所有的嵌入式容器定制器，调用定制器的定制方法

#### 5.嵌入式Servlet的启动原理

什么时候创建嵌入式的Servlet容器工厂？什么时候获取嵌入式的Servlet容器工厂？

获取嵌入式的Servlet容器工厂：

1. Springboot应用启动run方法

2. refreshContext（context);springboot刷新ioc容器，并初始化容器，创建容器中的每一个组件；

3. refresh(context) ;刷新创建号的ioc容器

4. onrefresh(),web的ioc容器重写了onRefresh方法

5. webIoc容器会创建嵌入式的Servlet容器

6. 获取嵌入式的Servlet容器工厂，从Ioc容器中获取EmbededServletContaionerFactory组件，创建Tomcat相应的容器对象，后置处理器一看是这个对象，就获取所有的定制器来定制Servlet容器的相关配置

7. 使用容器工厂获取嵌入式的servlet容器；

8. 嵌入式的Servlet容器创建对象并启动Servlet容器

   先启动嵌入式的Servlet容器，再将Ioc没有创建的对象获取处理

   **IOC容器启动时创建嵌入式Servlet容器**

###    9.使用外置的Sevlet容器

嵌入式Servlet容器：jar

​	优点：简单、便携；

​	缺点：默认不支持jsp，优化定制比较复杂

外置的Sevlet容器：war

步骤：

1. 必须创建一个war项目（创建目录结构）

2. 将嵌入式的Tomcat指定为provided

   ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
   </dependency>
   ```

3. 必须编写一个springbootServletInitializer的子类，并调用configure方法

   ```java
   public class ServletInitializer extends SpringBootServletInitializer {
   
   	@Override
   	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
           //传入主程序
   		return application.sources(SpringbootWarApplication.class);
   	}
   }
   ```

4. 启动服务器即可使用

**原理**

jar包：执行springboot主类的main方法，启动ioc容器，创建嵌入式的Servlet容器

war包：启动服务器，服务器启动springboot应用，才能启动ioc容器

servlet3.0（Spring注解版）：

**规则**：

1. 服务器启动（web应用启动）会创建当前web应用里面的每一个jar包里面有ServletContaionerInitilizer实例
2. ServletContaionInitilizer的实现放在jar包的META-INF/services文件夹下,有一个名字为javax.servlet.ServletContaionerInitializer的文件，内容就是ServletContaionerInitializer的实现类的全类名
3. 还可以使用@HandlesTypes,在应用中加载我们感兴趣的类

**流程**：

1. 启动服务器

2. spring-web/5.2.1.RELEASE/spring-web-5.2.1.RELEASE.jar!/META-INF/services/javax.servlet.ServletContainerInitializer的web模板有这个文件；内容是：**org.springframework.web.SpringServletContainerInitializer**

3. SpringServletContainerInitializer将@HandlesTypes({WebApplicationInitializer.class})标注的所有这个类型的类都传入到onstartup的方法的set中；为这些类WebApplicationInitializer类型的类创建实例

4. 每一个实例调用自己的onstartup();

5. 相当于我们的SpringBootServletInitializer的类会被创建对象，并执行onstartup方法

6. SpringBootServletInitializer执行时会创建createRootApplicationContext；创建容器

   ```java
   protected WebApplicationContext createRootApplicationContext(ServletContext servletContext) {
       SpringApplicationBuilder builder = this.createSpringApplicationBuilder();
       builder.main(this.getClass());
       ApplicationContext parent = this.getExistingRootWebApplicationContext(servletContext);
       if (parent != null) {
           this.logger.info("Root context already created (using as parent).");
           servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, (Object)null);
           builder.initializers(new ApplicationContextInitializer[]{new ParentContextApplicationContextInitializer(parent)});
       }
   
       builder.initializers(new ApplicationContextInitializer[]{new ServletContextApplicationContextInitializer(servletContext)});
       builder.contextClass(AnnotationConfigServletWebServerApplicationContext.class);
       //调用configure方法，子类重写了这个方法，将我们的主程序类传入
       builder = this.configure(builder);
       builder.listeners(new ApplicationListener[]{new SpringBootServletInitializer.WebEnvironmentPropertySourceInitializer(servletContext)});
       //使用builder创建一个spring应用
       SpringApplication application = builder.build();
       if (application.getAllSources().isEmpty() && MergedAnnotations.from(this.getClass(), SearchStrategy.TYPE_HIERARCHY).isPresent(Configuration.class)) {
           application.addPrimarySources(Collections.singleton(this.getClass()));
       }
   
       Assert.state(!application.getAllSources().isEmpty(), "No SpringApplication sources have been defined. Either override the configure method or add an @Configuration annotation");
       if (this.registerErrorPageFilter) {
           application.addPrimarySources(Collections.singleton(ErrorPageFilterConfiguration.class));
       }
   	//执行spring应用
       return this.run(application);
   }
   ```

7. spring的应用就启动了，并且创建Ioc容器

## 4.Docker

---

### 1.简介

**Docker是一个开源的应用容器引擎**，核心是：Docker支持将软件编译成一个镜像，然后在镜像中各种软件做好配置，将镜像发布出去，其他使用者可以直接使用这个镜像。运行中的这个镜像称为容器，容器启动是快速的。

### 2.Docker的核心概念

docker主机（Host)：安装了Docker程序的机器（Docker直接安装在操作系统上）

docker客户端（client）：客户端通过命令行或者其他工具使用Docker，与Docker的守护进程通信

docker容器（Container）：镜像启动后的实例就是容器，容器是独立运行的一个或一组应用

docker仓库（Registry）：Docker仓库用来保存镜像

docker镜像（Images）：软件打包好的镜像，放在docker仓库中

**步骤**：

1. 安装Docker
2. 去仓库找到这个软件对应的镜像
3. 使用Docker运行这个镜像，这个镜像就会生成一个Docker容器

### 3.使用Docker

#### 1.基本操作

启动docker

systemctl start docker

查找相应镜像

docker search mysql

拉取镜像

docker pull mysql:tags 默认是最新的

查看镜像

docker images

删除镜像

docker rmi image-id

#### 2.容器操作

步骤

```shell
1.查找tomcat
docker search tomcat
2.拉取镜像
docker pull tomcat
3.根据镜像启动容器 -name是自定义容器名 -d后台运行
docker run --name container-name -d image-name
docker run --name mytomcat -d tomcat:lastest
4.查看哪些容器启动列表
docker ps 查看运行中的容器
5.停止运行中的容器
docker stop container_id或image
6.查看所有容器
docker ps -a
7.启动容器
docker start container_id
8.删除容器（必须是容器是停止状态）
docker rm container_id
9.端口映射 8888是主机端口 8080是容器端口
docker run --name mytomcat -d -p 8888:8080 tomcat:lastest
10.关闭防火墙 centos
service firewalld status
service firewalld stop
11.查看容器日志
docker logs containerid/containName
```

#### 3.安装mysql实例

```shell
docker pull mysql
#错误的启动 
docker run --name  mysql -d mysql
#正确启动
docker run --name mysql01 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
docker run --name mysql01 -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

## 5.数据访问

### 1.简介

对于数据访问层，无论是sql还是nosql，springboot模式使用springdata的方式进行统一处理，添加大量自动配置，屏蔽很多配置。引入各种xxxtemplate，xxxRepository来简化我们对数据访问层的操作

-JDBC

-Mybatis

-JPA

### 2.JDBC

配置

```yml
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://192.168.15.22:3306/jdbc
    driver-class-name: com.mysql.jdbc.Driver
```

效果：

​	默认使用org.apache.tomcat.jdbc.poo.DataSource作为数据源

​	数据源的相关配置都在DataSourceProperties里面

原理：

1. 参考DataSourceConfiguration，根据配置创建数据源，默认使用tomcat连接池；可以使用spring.datasource.type指定自定义的数据源类型

2. springboot默认可以支持多种

3. 自定义数据源类型

   ```java
   @Configuration(
       proxyBeanMethods = false
   )
   @ConditionalOnMissingBean({DataSource.class})
   @ConditionalOnProperty(
       name = {"spring.datasource.type"}
   )
   static class Generic {
       Generic() {
       }
   
       @Bean
       DataSource dataSource(DataSourceProperties properties) {
           //使用build构建，利用反射创建相应type的数据源，并且绑定相关属性
           return properties.initializeDataSourceBuilder().build();
       }
   }
   ```

4. DataSourceInitilizer:ApplicationListener

   作用：

   1. runSchemaScripts()，运行建表语句
   2. runDataScripts()：运行插入数据

   默认只需将文件命名为：(classpath下)

   ```properties
    schema-*.sql  data-*.sql
    默认规则：schema.sql 
   ```

   自定义规则

   ```yml
   spring:
     datasource:
       username: root
       password: 123456
       url: jdbc:mysql://192.168.15.22:3306/jdbc
       driver-class-name: com.mysql.jdbc.Driver
       schema:
         - classpath:department.sql
         #指定位值
   ```

5. 访问数据 JdbcTemplate

### 3.配置druid数据源

配置

```yml
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://192.168.15.22:3306/jdbc
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
```

配置druid其他属性需要配置DataSource,以及Druid监控

```java
@Configuration
public class DruidConfig{
    @ConfigurationProperties(prefix="spring.datasource")
    @Bean
    public DataSource druid(){
        return new DruidDataSource();
    }
	//配置Druid的监控
	//1.配置一个管理后台的servlet
	@Bean
	public ServletRegistrationBean statViewServlet(){
		ServletRegistrationBean bean=new ServletRegistrationBean(new StatViewServlet(),"/druid");
		Map<String,String> map =new HashMap<>();
		map.put("loginUsername","root");
		map.put("loginPassword","666666");
		map.put("allow","");//默认就是允许所有
		bean.setInitParameters(map);
		return bean;
	}
	//2.配置一个web监控的filter
	@Bean
	public FilterRegistrationBean webStatFilter(){
		FilterRegistrationBean bean=new FilterRegistrationBean();
		bean.setFilter(new WebStatFilter());
		Map<String,String > map=new HashMap<>();
		map.put("exclusions","*.js,*.css,/druid/*");
		bean.setInitParameters(map);
		bean.setUrlPatterns(Arrays.asList("/*"));
		return bean;
	}
}
```

### 4.Mybatis

引入依赖

```xml
  <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-starter</artifactId>
      <version>2.1.1</version>
</dependency>
```

#### 1.注解

```java
//指定这是一个操作数据库的mapper,或者在主类配置@MapperScan批量扫描到容器中
@Mapper//必须项 2选1
public interface EmployMapper {
	//@Insert() @Delete() @Update()
	@Select("select * from employee where id = #{id}")
	public Employee getEmp(Integer id);
    //返回id
    @Options(useGeneratedKeys=true,keyProperty="id")
    @Insert("insert into employee(username,password) values(#{username},#{password})")
    public int insertEmp(Employee employee);
}
```

问题

执行驼峰映射 departmentName --- department_name

```java
@org.springframework.context.annotation.Configuration
public class MyBatisConfig {
	@Bean
	public ConfigurationCustomizer configurationCustomizer(){
		return new ConfigurationCustomizer() {
			@Override
			public void customize(Configuration configuration) {
				configuration.setMapUnderscoreToCamelCase(true);
			}
		};
	}
}
```

#### 2.xml方式

配置

```yml
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classspath:mybatis/mapper/*.xml
```

开启驼峰命名规则

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
<settings>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
</configuration>
```



### 5.Jpa

#### 1.概述

##### 1.springdata特点

springdata为我们提供使用统一的api来对数据访问层进行操作；这主要是springdata commons项目来实现的。spring data commons让我们在使用关系型和非关系型数据时都基于spring提供的统一标准。

##### 2.统一的repository接口

repository<T,ID extends Serializable>:统一接口

CrudRepository<T,ID extends Serializable>:基本crud

PagingAndSortingRepository<T,ID extends Serializable>：基本分页和crud

##### 3.提供数据访问模板类

如：RedisTemplate

#### 2.整合JPA(JAVA PERSISTENCE API)

Jpa:ORM(Object Relational Mapping)

1. 编写一个实体类（bean）和数据映射

```java
//使用jpa注解配置映射关系
@Entity//告诉Jpa这是一个实体类（和数据表映射的类）
@Table(name="tb_customer")//@Table来指定和哪一个数据表对应；如果省略表名默认为类名小写
public class Customer {
	@Id//这是一个注键
	@GeneratedValue(strategy = GenerationType.IDENTITY)//自增主键
	private Integer id;
	@Column(name = "name",length = 50)//省略默认列名就是属性名
	private String name;
}
```

2. 编写一个Dao接口来操作实体类

```java
//1.实体类类型 2.主键类型
//继承JpaRepository来完成
public interface UserRepository extends JpaRepository<Customer,Integer> {

```



3. 基本的配置

```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/java
    username: root
    password: 666666
  jpa:
    hibernate:
      #更新或者创建数据表结构
      ddl-auto: update
      # 显示sql语句
    show-sql: true
```

4. controller

```java

@RestController
public class CustomerController {
	@Autowired
	private CustomerRepository customerRepository;
	@GetMapping(value = "/custs")
	private List<Customer> getAll(){
		return customerRepository.findAll();
	}
	//为了方便
	@GetMapping(value = "/cust")
	private Customer insertCust(Customer customer){
		Customer save = customerRepository.save(customer);
		return save;

	}
}
```

## 6.启动配置原理

几个重要的事件回调机制

配置在META-INF/spring.factories

**ApplicationContextInitializer**

**SpringApplicationRunListener**

只需要放在ioc容器中

ApplicationRunner

CommandLineRunner

启动流程：

1. 创建springApplication对象

```java
//加载所有initializer
this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
//加载所有listener
this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
//在多个配置类中找到有main方法的配置类
this.mainApplicationClass = this.deduceMainApplicationClass();

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
    //此处调用下面方法
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
 private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        int var10 = var9.length;

                        for(int var11 = 0; var11 < var10; ++var11) {
                            String factoryImplementationName = var9[var11];
                            result.add(factoryTypeName, factoryImplementationName.trim());
                        }
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var13) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
            }
        }
    }

private static <T> T instantiateFactory(String factoryImplementationName, Class<T> factoryType, ClassLoader classLoader) {
    try {
        Class<?> factoryImplementationClass = ClassUtils.forName(factoryImplementationName, classLoader);
        if (!factoryType.isAssignableFrom(factoryImplementationClass)) {
            throw new IllegalArgumentException("Class [" + factoryImplementationName + "] is not assignable to factory type [" + factoryType.getName() + "]");
        } else {
            return ReflectionUtils.accessibleConstructor(factoryImplementationClass, new Class[0]).newInstance();
        }
    } catch (Throwable var4) {
        throw new IllegalArgumentException("Unable to instantiate factory class [" + factoryImplementationName + "] for factory type [" + factoryType.getName() + "]", var4);
    }
}
```

```properties
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```



2. 运行run方法

```java
    public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        //获取springapplicationrunlistener；从类路径下meta-inf/spring.factories下找
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        //回调所有的获取springapplicationrunlistener.starting方法
        listeners.starting();

        Collection exceptionReporters;
        try {
            //封装命令行参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //准备环境
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            //创建环境后回调SpringApplicationRunListener.prepareEnviroment（）方法,环境准备完成
            this.configureIgnoreBeanInfo(environment);
            //打印图标
            Banner printedBanner = this.printBanner(environment);
            //创建applicationContext；觉得是web的ioc容器还是普通的ioc容器
            context = this.createApplicationContext();
            //异常报告
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            //准备上下文环境；将environment保存到ioc中；而且applyInitializer（）,回调之前所有的springapplicationcontextInitializer的initialize方法；
            //回调所有的springapplicationrunlistener的contextprepared方法
            //preparecontext运行完成后回调所有SpringApplicationRunListener的contextloaded方法
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //刷新容器：ioc容器初始化；如果是web应用还会创建嵌入式的tomcat
            //扫描，创建，加载所有组件的地方（配置类，组件，自动配置）
            this.refreshContext(context);
            //从ioc容器中获取所有的applicationrunner和commandLineRunner进行回调
            //applicationrunner先回调，commandLinerunner再回调
            //所有的springapplicationrunlistener的finished方法
            this.afterRefresh(context, applicationArguments);
            //保存状态
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
            //返回ioc容器
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }

```

## 7.自定义starter

starter：

1. 这个场景需要使用的依赖是什么
2. 如何编写自动配置

```java
@configuraion //指定这个类是一个配置类
@conditionalonXXX //指定在什么条件下有效
@autoconfiguraionAfter //指定自动配置类的顺序
@Bean //给容器中添加组件
@ConfiguraionProperties //结合相关XXXProperties绑定相关配置
@EnableConfiguationProperties //使XXXProperties生效自动加入到容器中
自动配置类需要加载
将需要启动就加载的自动配置类，配置在META-INF/spring.factories下
```

3. 模式

启动器只用来做依赖导入

专门来写一个自动配置模块；

启动器依赖自动配置；别人只需要依赖启动器

mybatis-spring-boot-starter:自定义

spring-boot-starter;所有starter的基本配置