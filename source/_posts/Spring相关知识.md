---
title: Spring相关知识
date: 2017-07-24 11:50:56
tags: [spring,annotation]
---

## 传统MVC控制器与RESTful Web Service控制器的不同
> 传统MVC控制器返回的是一个数据的视图（Model），而RESTful Web Service控制器返回的是Json格式的对象
<!--more-->

---

## * Spring的注解
>   * @component 泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注，作用是将对象实例化到Spring容器中
>   * @Service用于标注业务层组件；@Controller用于标注控制层组件（如struts中的action）；@Repository用于标注数据访问组件，即DAO组件，使用jpa访问数据库时，Repository类实现了很多数据库查询操作，方便用户进行数据库操作。
>   * @componentscan标签默认情况下自动扫描指定路径下的包（含所有子包），将带有@Component、@Repository、@Service、@Controller标签的类自动注册到spring容器。对标记了 Spring的 @Required、@Autowired、JSR250的 @PostConstruct、@PreDestroy、@Resource、JAX-WS的 @WebServiceRef、EJB3的 @EJB、JPA的 @PersistenceContext、@PersistenceUnit等注解的类进行对应的操作使注解生效（包含了annotation-config标签的作用）
>   * @Autowired可以对类成员变量、方法及构造函数进行标注，完成自动装配的工作。 通过 @Autowired的使用来消除 set ，get方法
>   * @Order可以使数组有序，指定@Order(number)即可设定该Bean在数组中的排序值
>   * @Configuration标注在类上，相当于把该类作为spring的xml配置文件中的<beans>，作用为：配置spring容器(应用上下文)
>   * @Bean标注在方法上(返回某个实例的方法)，等价于spring的xml配置文件中的<bean>，作用为：注册bean对象
>   * @Valid配合JSR303定义的校验类型（@NotNull，@Min等），并紧挨着一个BindingResult 参数，可以完成对Bean对象的自动校验

---
## AOP（面向切面编程）
>   * Aspect（切面）：声明一个类为切面，并为该切面设置切入点（pointcut）后，切入点扫描某一个包中的某一个类中的某一个方法，在该方法被执行的时候，可以指定切面中的某一方法伴随该方法执行，可以是该方法执行之前，也可以是之后。扫描的方式可以是：
```
expression="execution(* com.xrq.aop.HelloWorld.*(..))
```


