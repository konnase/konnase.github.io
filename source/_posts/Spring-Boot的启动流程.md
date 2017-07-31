---
title: Spring Boot的启动流程
date: 2017-07-25 15:23:07
tags:
---

# 1. 程序启动入口

```
@SpringBootApplication
public class WeatherApplication {

    public static void main(String[] args) {
        SpringApplication.run(WeatherApplication.class, args);
    }
}
```
执行SpringApplication的静态run方法，将WeatherApplication类和main方法的args作为参数传进去。
<!--more-->

在下面的RestController类中定义了Web端访问的API，当访问的URL的path部分是"/"时，返回"Hello World!"

```
@RestController
public class WeatherController {

    @RequestMapping("/")
    public String byIndex() {
        return "Hello world！";
    }

}
```
# 2. SpringApplication类的静态run方法的调用

该方法位于org.framework.boot.SpringApplication类中：

```
public static ConfigurableApplicationContext run(Object source, String... args) {
    return run(new Object[] { source }, args);
}

public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    return new SpringApplication(sources).run(args);
}
```
从第一步中传递进来的对象和main方法参数匹配到以上代码中第一个静态run方法，该方法创建对象数组并调用第二个静态run方法，根据传入的对象数组新建SpringApplication对象并调用其非静态run方法（该方法定义将在后面章节进行介绍）。执行完后Spring容器就启动起来了。

# 3.新建SpringApplication对象
