---
title: Maven学习笔记
p: java_spring/Maven学习笔记
date: 2017-11-05 12:59:16
tags: [maven,spring]
categories: java
---

## TheNEXUS Maven By Example

### Question
1. 5.6节提到的依赖中设置scope为provided：This tells Maven that the JAR is “provided” by the container and thus should not be included in the WAR. 是什么意思？
<!--more-->
2. 第六章结合了Weather Module和Web Module，其中，在Web Module中加入了Weather的依赖，并添加了WeatherService服务，该服务中的retrieveFormat方法同Main中的start方法完成相同的工作。Web Module中新增WeatherServlet，里面的doGet方法完成实现对http请求的处理，请求天气数据并返回结果到页面展示。
3. 当Maven执行一个有子模块的project的时候，Maven首先加载parent POM，然后定位所有的子模块。然后将所有的POM交给Maven Reactor，负责分析模块之间的依赖，确保以相应的顺序编译和安装相互依赖的模块。
4. 第六章运行 mvn jetty:run的时候，提示找不到simple-weather-08-SNAPSHOT.jar这个包。而且Yahoo的weather.yahooapis.com网站访问不了。
5. web.xml和weather-servlet.xml都是自动加载的吗？

### 第七章对象关系
![image](/img/multimodule-web-spring_projects.png)

* SimpleModel：由Weather、Condition、Atmosphere、Location、Wind对象组成。主要是向simple-weather提供Weather对象，simple-wather解析XML后会生成Weather对象。在simple-persist的DAO中将Weather对象存储到数据库中。
* simple-weather：POM文件中加入对simple-model的依赖，simple-weather中定义WeatherService，通过YahooRetriever和YahooParser从Yahoo取得天气信息，返回Weather对象。simple-webapp和simple-command中通过applicationContext-weather.xml中定义的Bean–weatherService来取得该weatherService。
* simple-persist：POM文件中加入对simple-model的依赖。LocationDAO和WeatherDAO分别实现从数据库中通过zip code查找location信息和通过location查找Weather信息。applicationContext-persist.xml中向Spring容器中注入了locationDAO和weatherDao，simple-webapp和simple-command中调用这两个Bean完成数据库查询工作。
* simple-webapp：POM中加入对simple-weather和simple-persist的依赖，定义WeatherController从simple-weather获取Weather对象的数据，定义HistoryController从simple-persist获取数据库查询结果。weather-servlet.xml注册了两个controller，并定义了urlMapping以及限定了Velocity模板引擎访问资源的目录。
* simple-command：同时依赖于simple-weather和simple-persist，执行Main方法，根据传入的参数的选择执行请求Weather或者查询相应zip code对应的城市的天气情况。
* 这种设计模式，将对象模型、数据持久层、业务层（service）、逻辑层（webapp和command）分开，五个submodule各司其职，相互之间存在依赖关系，利于项目的管理。