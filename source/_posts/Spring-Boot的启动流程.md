---
title: Spring Boot的启动流程
date: 2017-07-25 15:23:07
tags: [spring boot,spring]
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

在下面的RestController类中定义了Web端访问的API，当访问的URL的path部分是"/"时，返回"Hello World!"

<!--more-->

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
SpringApplication对象的构造方法中调用initialize方法，主要为SpringApplication对象赋一些初值。

# 进入run方法
``` java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        this.configureHeadlessProperty();
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            Banner printedBanner = this.printBanner(environment);
            context = this.createApplicationContext();
            new FailureAnalyzers(context);
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            listeners.finished(context, (Throwable)null);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, listeners, (FailureAnalyzers)analyzers, var9);
            throw new IllegalStateException(var9);
        }
    }
```
1. 创建应用的监听器SpringApplicationRunListeners并开始监听

2. 加载SpringBoot配置环境(ConfigurableEnvironment)，如果是通过web容器发布，会加载StandardEnvironment，其最终也是继承了ConfigurableEnvironment
3. 配置环境(Environment)加入到监听器对象中(SpringApplicationRunListeners)

4. 创建run方法的返回对象：ConfigurableApplicationContext(应用配置上下文)
其中，createApplicationContext方法会先获取显式设置的应用上下文(applicationContextClass)，如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择AnnotationConfigApplicationContext注解上下文（通过扫描所有注解类来加载bean），最后通过BeanUtils实例化上下文对象，并返回。

5. 
