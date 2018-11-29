---
title: Spring相关知识2
date: 2018-11-14 19:43:48
tags: [spring, jpa]
catagories: java
---

# JPA 和 Mybatis
jpa是一套ORM规范或接口（注意Mybatis没有实现jpa），默认使用Hibernate作为ORM实现，所以一般使用Spring Data JPA即会使用Hibernate。jpa的查询语言是面向对象的而不是面向数据库的。Hibernate是一个ORM框架，它对JDBC进行了非常轻量级的对象封装，将POJO与数据库表建立映射关系，可以自动生成sql语句，使得Java程序员可以从繁琐的sql代码中解脱出来

DAO（数据库访问对象），在jpa里面一般叫做Repository，比如UserRepository；在Mybatis里面一般叫Mapper，比如UserMapper

操作数据库的对象：Hibernate中叫session，jpa中叫entityManager，Mybatis中叫SqlSession

# Mybatis
SqlSessionFactory负责创建SqlSession，其作用域应该设置为整个应用，生命周期为应用运行时；SqlSession的实例不是线程安全的，因此不能被共享，所以其作用域是请求或者方法作用域，绝不能将SqlSession实例的引用放在一个类的静态域中，甚至一个类的实例变量都不行。如果处理一个Http请求，打开一个SqlSession，处理完成之后应立即关闭SqlSession，可以放到finally语句中执行

Mybatis支持动态Sql查询，即根据sql语句中前半部分的结果决定是否执行后半部分的查询或者条件
@Mapper注解：我猜这个注解在该mapper被调用时会自动创建一个SqlSession，执行完之后关闭这个SqlSession

# Spring data jpa
按照Spring Data JPA的规则自动生成sql语句，无需任何sql语句即可实现CRUD

比如findByNameAndPassword(String name, String password)方法，spring识别findBy作为: “select * from XXX”，sql语句的where部分则是: “where name = :name and password = :password”
参考： https://docs.spring.io/spring-data/jpa/docs/current/reference/html/

