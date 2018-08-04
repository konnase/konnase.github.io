---
title: spring-boot热部署
date: 2018-01-20 16:27:03
tags: [spring boot, hot deploy]
categories: JAVA
---

## spring-boot热部署
1. 第一种方式
在pom文件中的plugin下添加：
```
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>springloaded</artifactId>
        <version>1.2.6.RELEASE</version>
    </dependency>
</dependencies>
```

2. 第二种方式
直接在pom文件中引入依赖：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools/artifactId>
    <optional>true</optional>
</dependency>
```
