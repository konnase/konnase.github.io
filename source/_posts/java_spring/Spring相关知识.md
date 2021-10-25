---
title: Spring相关知识
p: java_spring/Spring相关知识
date: 2017-07-24 11:50:56
tags: [spring,annotation,spring mvc]
categories: java
---

## 传统MVC控制器与RESTful Web Service控制器的不同
 传统MVC控制器返回的是一个数据的视图（Model），而RESTful Web Service控制器返回的是Json格式的对象

<!--more-->
---

## Spring的模块

![image](/img/spring-overview.png)

* 上图中每个小单元都至少对应一个jar包，提供相应的API供使用。使用Maven工具管理Spring项目时，这些需要用到的jar包集中在用户目录下的.m2文件夹下，而不是包含在项目中。
*  7个模块
> 1. Spring Core：核心容器的主要组件是BeanFactory，BeanFactory使用的是IoC模式
> 2. Spring Context：向Spring框架提供上下文信息的一个配置文件
> 3. Spring AOP：面向切面的编程
> 4. Spring DAO：数据访问层的设计机制，DAO模式是业务逻辑层和持久存储层之间的抽象层，避免持久化代码和业务逻辑混合在一起。
> 5. Spring ORM：对象/关系映射（Object/Relation Mapping），指的是自动将java对象状态映射到关系数据库中的数据上。
> 6. Spring Web：为基于Web的应用程序提供上下文
> 7. Spring MVC：控制器将接收请求，执行更新模型的操作，然后通知视图关于模型更改的消息，依据请求的状态以及请求的控制器，可以决定显示哪个视图。


## Spring的注解
* @component 泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注，作用是将对象实例化到Spring容器中
* @Service用于标注业务层组件；@Controller用于标注控制层组件（如struts中的action）；@Repository用于标注数据访问组件，即DAO组件，使用jpa访问数据库时，Repository类实现了很多数据库查询操作，方便用户进行数据库操作。
* @componentscan标签默认情况下自动扫描指定路径下的包（含所有子包），将带有@Component、@Repository、@Service、@Controller标签的类自动注册到spring容器。对标记了 Spring的 @Required、@Autowired、JSR250的 @PostConstruct、@PreDestroy、@Resource、JAX-WS的 @WebServiceRef、EJB3的 @EJB、JPA的 @PersistenceContext、@PersistenceUnit等注解的类进行对应的操作使注解生效（包含了annotation-config标签的作用）
* @Autowired可以对类成员变量、方法及构造函数进行标注，完成自动装配的工作。 通过 @Autowired的使用来消除 set ，get方法
* @Order可以使数组有序，指定@Order(number)即可设定该Bean在数组中的排序值
* @Configuration标注在类上，相当于把该类作为spring的xml配置文件中的<beans>，作用为：配置spring容器(应用上下文)
* @Bean标注在方法上(返回某个实例的方法)，等价于spring的xml配置文件中的<bean>，作用为：注册bean对象
* @Valid配合JSR303定义的校验类型（@NotNull，@Min等），并紧挨着一个BindingResult 参数，可以完成对Bean对象的自动校验

## Spring的事件
为Bean与Bean之间的消息通信提供支持。Spring时间的流程如下：
1. 自定义事件，继承自ApplicationEvent
2. 定义事件监听器，实现ApplicationListener，并指定监听的事件类型，即第一步的自定义事件。
3. 使用容器发布事件

---
## AOP（面向切面编程）
* Aspect（切面）：声明一个类为切面，并为该切面设置切入点（pointcut）后，切入点扫描某一个包中的某一个类中的某一个方法，在该方法被执行的时候，可以指定切面中的某一方法伴随该方法执行，可以是该方法执行之前，也可以是之后。扫描的方式可以是：
```
expression="execution(* com.xrq.aop.HelloWorld.*(..))
```
* 好处是可以把应用业务逻辑和系统服务分开

## 控制反转（IoC）
* 对象们给出它们的依赖，而不是在代码中创建或查找依赖的对象们，即不需要new这些对象，而只需要描述如何创建这些对象。换种说法，依赖注入指的是容器负责创建对象和维护对象之间的依赖关系，而不是通过对象本身负责自己的创建和解决自己的依赖。
* 控制反转分为依赖注入（DI）和依赖查找（DL）
* Spring IoC负责创建对象、管理对象（DI）、装配对象并管理这些对象的整个生命周期


## 深入理解Spring
### Bean的实例化
* Spring容器启动的过程中，会将Bean解析成Spring内部的BeanDefinition结构，之后对Bean的操作就直接对BeanDefinition进行；BeanDefinition是一个接口，是一个抽象的定义，实际使用的是他的实现类。
* 然后以beanName为key，BeanDefinition为value存储到DefaultListableBeanFactory中的beanDefinitionMap（ConcurrentHashMap类型）中，DefaultListableBeanFactory间接实现BeanFactory接口，后者是用于访问Spring Bean容器的根接口，最常用的方法就是getBean方法。DefaultListableBeanFactory间接实现了DefaultSingletonBeanRegistry，后者有一个ConcurrentHashMap类型的属性singletonObjects，用于存储单例bean，getBean方法就是从这里获取实例对象。
* 将beanName存入beanDefinitionNames（List类型）中
* 遍历beanDefinitionNames，完成对bean的实例化并填充属性。
* 实例化后将实例存入单例bean的缓存中，当调用getBean方法时，到单例bean的缓存中查找并返回这个实例

### Bean的生命周期
* 创建Bean的实例
* 属性注入
* 初始化Bean
> * 激活Aware方法
> * 处理器应用
> * 激活自定义的init方法
* 使用Bean
* 销毁

## Spring MVC
![image](/img/springmvc.png)
1. 用户发送请求--> DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他解析器进行处理，作为统一访问点，进行全局流程控制。
2. DispatherServlet--> HandlerMapping，后者将把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略
3. DispatcherServlet--> HandlerAdapter，后者将会把处理器包装为适配器，从而支持多种类型的处理器
4. HandlerAdapter--> 处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理，并返回一个ModelAndView对象
5. ModelAndView的逻辑视图名--> ViewResolver，ViewResolver将把逻辑视图名解析成具体的View，通过这种策略，很容易更换其他视图技术
6. View--> 渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术
7. 返回控制权给DIspatcherServlet，由DispatcherServlet返回响应给用户，流程结束