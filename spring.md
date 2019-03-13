* spring的理解
    
    spring是以IOC和AOP为核心的众多组件的总称, 其他还包括spring-boot, spring-mvc等组件, 涵盖了后台开发的从前端到后台交互, 后台分层, 中间件, 数据库等持久层等几乎方位的支持.
### spring IOC
* 大致思想
首先spring的控制反转来源于依赖倒置(Dependency Inversion Principle )的思想, 
传统的依赖顺序是高层依赖底层, 而依赖倒置指的是底层依赖高层. 例如车子的建造, 车子依赖车身, 车身依赖底盘, 底盘依赖车轮, 若需要改动车轮的规格, 则以上所有东西都需要改, 若反过来, 先定好车子的样子, 车身 -> 车子, 底盘 -> 车身, 轮子 -> 底盘, 要改车轮, 不会影响其他.  

* 概念关系
依赖倒置是思想, 控制反转是实现该思想的一个思路, 依赖注入是该思路的具体实现方法, 依赖注入可以简要说是把底层类作为参数传递给上层类, 实现上层对下层的控制, 而IOC容器则封装了底层类的构造方法, 不需要知道底层类的实现细节以及 具体的构造函数的如何调用, 只需要按照依赖注入即可.


---
以下内容均参考 [spring-framework-reference/core/beans-annotation-config](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-annotation-config)

* xml和annotation两种方式比较

    annotation使配置更简洁, 和源码放在一起, 更精确, 但是配置比较分散, 不在一个地方很难控制 ; xml的方式不需要去动源码, 所以不需要重新编译, 
    
* @Required 和@Autowired 比较

    前者如果没有注入成功, 会报错, 且只能用于setter method.; 后者可以设置是否required(默认true) ,并且当无法autowire时会自动忽略, 适用于constructor, field , setter method or config method.
    
* @Autowired
    
    按照byType注入, 可以设置required=false, 找不到bean时也不报错, 后续可能报NPE.
    * 用于constructor, 当构造函数有多个时, 可以将指定一个进行autowired, 否则将默认以贪婪模式(参数最多的构造函数进行构造).

    * 获取所有类型的bean
    > It is also possible to provide all beans of a particular type from the ApplicationContext by adding the annotation to a field or method that expects an array of that type:
    
    ```
    public class MovieRecommender {

    @Autowired
    private MovieCatalog[] movieCatalogs;

    }
    
    public class MovieRecommender {

    private Set<MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
    }
    ```
* @Qualifier
    
    > 可以结合@Autowire使用, 当某个type有多个实例时, 可以用@Qualifier 指定某个id的bean注入. @Qualifier和@Autowired结合使用时, 和@Resource几乎等同. 例如

```

    @Autowired
	@Qualifier("personA")
	private Person person;
	
	<bean id="personA" class="com.mkyong.common.Person" >
		<property name="name" value="mkyongA" />
	</bean>
	
	<bean id="personB" class="com.mkyong.common.Person" >
		<property name="name" value="mkyongB" />
	</bean>

```
* @Resource
     
    既可以byName, 也可以byType注入, 默认byName, 就是按照bean的id注入.
    * 装配顺序
    1. 如果同时指定了name和type，则从Spring上下文中找到唯一匹配的bean进行装配，找不到则抛出异常

    2. 如果指定了name，则从上下文中查找名称（id）匹配的bean进行装配，找不到则抛出异常

    3. 如果指定了type，则从上下文中找到类型匹配的唯一bean进行装配，找不到或者找到多个，都会抛出异常

    4. 如果既没有指定name，又没有指定type，则自动按照byName方式进行装配；如果没有匹配，则回退为一个原始类型进行匹配，如果匹配则自动装配；
    
    *问题*  在细看一下@Qualifier
    
* @Component、@Repository、@Service、@Controller

    如果 Web 应用程序采用了经典的三层分层结构的话，最好在持久层、业务层和控制层分别采用上述注解对分层中的类进行注释。@Service用于标注业务层组件.@Controller用于标注控制层组件（如struts中的action）.@Repository用于标注数据访问组件，即DAO组件. @Component泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。
    
    * @Bean与@Component的区别
        
        前者是显式的返回一个object, 然后注入到application context, 让spring代管理, 例如可以适用于第三方的lib, 无法再源码上@Component; 后者是让spring创建并管理, 隐式的创建.


### classpath*: / 与 classpath:/
*  classpath*:/ 表示在此classpath(当前项目的classpath及类路径的classpath)下的所有兄弟和子孙路径的 classpath下满足要求的所有文件
*  classpath:/ 表示classpath下的第一层子路径的第一个符合要求的文件

### build project

* 新建maven project ，选择archetype 为webapp或site，例如 module 名为 test，module file location 和content root 均为test路径
* web项目下相对路径和绝对路径：

  * "." 代表目前所在的目录。
  * ".." 代表上一层目录。
  * "/" 代表根目录。

### 前后台传输数据


>1. 前台需要把对象通过JSON.stringfy转换json字符串，后台在参数对应加上@RequestBody,并且Request的content-type必须为json,不然会出现415的错误.

>2. 后台传输少量数据到前端,避免新增类文件的做法:

```
Map<String,Object> map = new HashMap<String,Object>();
map.put("leader",ifLeader);
map.put("teamId",teamId);
jsonResult.setData(new JSONObject(map));
```
3. 日期类参数
> * 如果controller的dto和后台数据映射dto为同一个，且日期类使用java.util.Date作为转换类型，则在该参数上加两个标签，省掉格式转换和非空判断,@DateTimeFormat为接受前台数据的转换，@JSONField为后台给前台返回数据的转换。

```
@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
@JSONField（format="yyyy-MM-dd HH:mm:ss")
Date startTime
```

> * spring-servlet.xml需要加以下配置

```
<mvc:annotation-driven>
        <mvc:message-converters register-defaults="true">
        <bean class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter">
            <property name="supportedMediaTypes" value="application/json"/>
            <!--设置fastjson特性-->
            <property name="features">
                <array>
                    <!--设置null值也要输出，fastjson默认是关闭的-->
                    <value>WriteMapNullValue</value>
                    <!--设置使用文本方式输出日期，fastjson默认是long-->
                    <value>WriteDateUseDateFormat</value>
                </array>
            </property>
        </bean>
        </mvc:message-converters>
    </mvc:annotation-driven>

```


### mybatis返回map
mapper:

```
<resultMap id="teamResultMap" type="com.mucfc.result.QueryTeamResult">
    <id column="id" property="teamId" jdbcType="INTEGER" javaType="int" />
    <result column="name" property="teamName"  jdbcType="VARCHAR" javaType="String"/>
</resultMap>
<select id="queryTeam"  resultMap="teamResultMap"  parameterType="int" >
    select id,name  from tb_dept WHERE parent= #{level}
</select>

// daoMybatis:
public List<QueryTeamResult> queryTeam(int level){
    return selectList("WeeklyMapper.queryTeam",level);
}
```

### jetty启动时修改资源文件,可以通过update更新

修改/${jetty_home}/etc/webdefault.xml 将true改为false
并在pom文件中配置
```
<init-param>
  <param-name>useFileMappedBuffer</param-name>
  <param-value>false</param-value>
</init-param>
```

```
pom.xml
<plugin>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-maven-plugin</artifactId>
  <configuration>
  <webAppConfig>
  <contextPath>/{project.build.finalName}</contextPath>
  </webAppConfig>
  <scanIntervalSeconds>3</scanIntervalSeconds>
  <httpConnector>8080</httpConnector>
  <webAppXml>jetty.xml</webAppXml>
  <webApp>
  <defaultsDescriptor>webdefault.xml</defaultsDescriptor>
  </webApp>
  </configuration>
</plugin>
```

### httpServletResquest

* getAttribute()
>An attribute is a server variable that exists within a specified scope i.e.:
application: available for the life of the entire application
session: available for the life of the session
request: only available for the life of the request
page (JSP only): available for the current JSP page only

* getParameter()
> returns http request parameters. Those passed from the client to the server. For example http://example.com/servlet?parameter=1. 
Can only return String

### spring @value
* static field cant inject directly.
```
// 类需要加@Component 标签
private static String test;
@Value("${test.val}")
public void setTest(String a){
    test = a;
}
```
* #{configProperties.configVal} , ${configVal} 区别
[link](http://blog.csdn.net/zengdeqing2012/article/details/50736119)

* 其他用法
[link](http://www.baeldung.com/properties-with-spring#usage)


## mybatis
* 搞清楚mybatis ${var}和#{var}的区别
#### Boolean 值传递
数据库类型是tinyInt, dto字段类型是Boolean或boolean, 

```
<if test="filedName != null">
 #{fieldName}
</if>
 // 不要加非空字符串判断
```


### spring AOP
> 代理技术可以说是封装了切面类(功能加强类)的逻辑和目标类的逻辑, 解耦两者的代码, 让功能加强类的逻辑能无缝插入目标类, 不产生浸入式代码.
因为以下两种原生代理, 存在以下几个问题: 
> 1. 不能对目标类的某些方法进行选择后进行切面
> 2. 还是通过硬编码的方式在代理类中, 在目标方法前后进行切面
> 3. 对每个类都要创建代理类, 无法实现通用; 因此spring aop应运而生. 
    

* pointcut 切点
    
    指定在哪些类和哪些方法做切入

* advice 增强
    
    定义加强类的逻辑, 方法前还是方法后等
    
* advisor 
    切面, 将pointcut和advice结合起来
 
* JDK动态代理
    只能对接口或者实现类进行方法切面, 
*问题* 
为什么只能基于接口做代理.

* CGLib代理

    采用动态创建子类的方法, 因此不能对final, private等方法进行切面; 而且相比于JDK性能比较高, 创建代理对象的时间比较长, 适用于spring 多数类都是单例的情况.

* 示例代码
```
// maven dependency
<!-- AspectJ dependencies -->
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjrt</artifactId>
			<version>${aspectj.version}</version>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjtools</artifactId>
			<version>${aspectj.version}</version>
		</dependency>
		
		
// spring application.xml config
<!-- Enable AspectJ style of Spring AOP -->
<aop:aspectj-autoproxy  />
// 下面这种方式用于CGLIB, 可以切面类的非接口方法, 否则上面那种基于JDK的代理只能切接口方法
<aop:aspectj-autoproxy  proxy-ttarget-class="true" />

@Aspect
// 注意需要依赖注入
@Component
public class EmployeeAspect {

	// 注意方法()内有两个点, 代表任意参数
	@Pointcut("execution(* com.journaldev.spring.service.*.get*(..))")
	public void pointCut(){
		System.out.println("enter pointcut");
	}
	
	@Before("pointCut()")
	public void before(){
		
	}
}
```

* 切面进入的条件
public , non static , non final ,called by class outside(非必须)
若要在类的内部调用中执行切面, 例如类里面有两个方法A, B, 在A调用B的时候进入切面, 则需要如下写法:

```
    // 注意该类需要由spring管理,并生成, 若是在外部 Test t = new Test(); t.methodA();或者 t.methodB(); 均不能进入切面
    @Component
    public class Test{
    @Resource Test test
        public void methodA(){
            // something
            test.methodB();
            // something
        }
        
        public void methodB(){}
        
    }
```



#### 事务传播行为（为了解决业务层方法之间事务互相调用的问题）
* require
> 如果当前没有事务, 则开启新事务, 否则会使用当前事务

* require_new
> 都会开始新事务, 适用于嵌套事务的情况, 如果a事务里面调用了一个b事务, 想让b中异常了, 但是a还是能正常提交, 则需要开启一个新事务, 则用require_new .

#### 事务回滚
> By default all RuntimeExceptions rollback transaction whereas checked exceptions don't. This is an EJB legacy. You can configure this by using rollbackFor() and noRollbackFor() annotation parameters:

```
@Transactional(rollbackFor=Exception.class)
This will rollback transaction after throwing any exception 
```

#### bean的生命周期
![181453414212066](6713CD4BADC64913AB494E6F2FCFF3BE)
> 包括以下几类

1. bean自身的方法, 定义的init和destroy方法
2. bean通用的生命周期的接口方法, 这个包括了BeanNameAware、BeanFactoryAware、InitializingBean和DiposableBean这些接口的方法
3. 容器级生命周期接口方法, 这个包括了InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个接口实现，一般称它们的实现类为“后处理器”。

**问题**
* mysql的锁
* spring的事务传播级别, 实际上就是多个方法调用, 发生多个事务交叉时, 如何选择当一个事务提交或者回滚后对另一个事务的影响


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU4ODg5OTkyMSwtMTI4Njc4OTE1OCwtNj
g5OTg5NjEzLC0xMTA2MTQzNjQ3LDE5OTczMTkxNjFdfQ==
-->