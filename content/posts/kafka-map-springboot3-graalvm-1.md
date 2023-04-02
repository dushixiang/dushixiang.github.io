---
title: "SpringBoot 3.0.0尝鲜与Java打包原生二进制【一】"
categories: [ "Java" ]
tags: [ "kafka-map", "SpringBoot", "graalvm" ]
draft: false
slug: "kafka-map-springboot3-graalvm-1"
date: "2022-11-26T23:44:00+08:00"
---

2022年11月24日 SpringBoot 正式发布了 3.0 版本，带来许多新的特性，但我最关心的还是Java打包成原生二进制，运行时不再依赖jre环境，运行Java程序将和Go程序一样方便。

## 升级至 SpringBoot 3.0.0

说是尝鲜，但是我不想再试着搞 hello world 那种啥都没有的东西了，找到我之前写的一个Java开源项目 [kafka-map](https://github.com/dushixiang/kafka-map) 拿他开刀。

[kafka-map](https://github.com/dushixiang/kafka-map) 本身是基于 SpringBoot 2.4.x 开发的，sqlite 存储数据，且很久没有大的更新了，想要直接升级到 SpringBoot 3.0.0 是不可能的，我按照官方文档 [Spring Boot 3.0 迁移指南](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide) 首先升级到最新2.7.x版本，然后就发现 service 依赖循环了，这个时候有两种选择，一是在配置文件中允许依赖循环 `spring.main.allow-circular-references: true`，二是梳理业务逻辑解决依赖循环的问题。作为一个合格的开发，我选择了解决依赖循环的问题，过程不表。

SpringBoot 3.0.0 升级了很多组件，其中 Jpa 依赖的 Hibernate 升级到了 6.x，我启动时又遇到了 sqlite 方言插件不可用的问题，还好 Hibernate 6.x 已经支持了 sqlite方言，切换到官方插件就好了。配置文件如下：

``` yaml
spring:
  datasource:
    url: jdbc:sqlite:data/kafka-map.db
    driver-class-name: org.sqlite.JDBC
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.community.dialect.SQLiteDialect
```

## 原生二进制打包

打包原生二进制还是最折腾的，刚开始参考[GraalVM Native Image Support](https://docs.spring.io/spring-boot/docs/3.0.0/reference/html/native-image.html#native-image.developing-your-first-application.buildpacks) 把打包 `springboot:build-image` 当成了打包原生二进制了，而且打包的过程中还遇到了 `UnsupportedFeatureException: No instances of ch.qos.logback.classic.Logger` 这个问题，找了一圈也没找到解决方案。

![image.png](https://oss.typesafe.cn/k5.png)

无奈，还是从 hello world 开始学起吧，我重新创建了一个 SpringBoot3.0.0 项目，并添加了 GraalVM Native Support 和我用到的一些组件。

![image.png](https://oss.typesafe.cn/k6.png)

然后我就发现了新的大陆，我之前的方向错了，果然是方向错了再努力也没用啊。

![image.png](https://oss.typesafe.cn/k4.png)

我在这个hello world程序中试了一下果然能行，当然是需要手动安装 GraalVM 并且配置环境变量的，切换到 graalvm 环境的jdk 。请参考 https://www.graalvm.org/downloads/ 的教程。

对比 hello world 程序我发现在 maven 插件中多了一个插件，复制过去安装一下。

``` xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

运行打包命令。

> 不要忘记激活 graavm 环境。
```
mvn clean native:compile -Pnative -Dmaven.test.skip=true
```

然后时长达十几分钟的等待。

![image.png](https://oss.typesafe.cn/k1.png)

运行成功？不，我又失败了。当然，我失败的很快，87毫秒。

![image.png](https://oss.typesafe.cn/k2.png)

![image.png](https://oss.typesafe.cn/k3.png)

查看原因，原来是找不到我的 sqlite 方言类，此时我突然想到 [GraalVM Native Image Support](https://docs.spring.io/spring-boot/docs/3.0.0/reference/html/native-image.html#native-image.developing-your-first-application.buildpacks) 其中一段话。

> Unlike traditional applications written for the JVM, GraalVM Native Image applications require ahead-of-time processing in order to create an executable. This ahead-of-time processing involves statically analyzing your application code from its main entry point.


> 翻译：与为JVM编写的传统应用程序不同，GraalVM Native Image应用程序需要提前处理才能创建可执行文件。这种超前处理涉及从应用程序的主要入口点静态分析应用程序代码。

GraalVM 从程序入口分析应用程序代码，而我在代码任何一个地方都没有引用过 `org.hibernate.community.dialect.SQLiteDialect` 这个类，然后它就被 GraalVM 无情的抛弃了。

## 总结

使用 GraalVM 打包Java程序为原生二进制确实为Java程序提供了极快的启动速度，无需jre运行环境，很方便，但之前开发的程序想要使用此特性恐怕还是有点困难，新开发的项目也需要摒弃掉之前的一些开发习惯才能使用，并且二进制文件太大、GraalVM 尚未支持交叉编译，对比Golang还有很大的差距。

因为我还尚未成功运行起原生二进制的 kafka-map，不清楚 GraalVM 对一些运行时的API 支持如何，后续我将会努力运行起来 原生二进制的 kafka-map，并且对功能进行详细的测试。

未完待续...



