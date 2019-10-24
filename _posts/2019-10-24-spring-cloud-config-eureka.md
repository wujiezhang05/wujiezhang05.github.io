---
title: spring cloud 配置中心 config + 服务发现注册中心 eureka
excerpt: spring cloud config 和 netflix eureka是spring cloud很重要的组件，通过一个例子来学习他们。
category: Java
tags: spring
---

 > spring cloud: 整合了一系列成熟的微服务框架，并用springboot的方式进行封装，
 > 屏蔽掉复杂的实现配置过程。从而可以低成本的搭建高效，分布式，容错微服务平台。
 > 通过一个例子来了解下spring cloud config 和 spring cloud eureka


- spring cloud config
  <br>配置中心：
  分布式开放下，配置文件会随着服务增大而增多。每个服务配置变化会引起一系列的更新重启。配置中心可以很好的解决此类问题。并提供更为强大的功能: 集中管理各环境下的配置文件，有版本控制，配置文件修改可以快速生效。
- spring cloud netflix eureka
  <br>注册中心：管理微服务的注册、发现、熔断、负载、降级，微服务之间不直接关联，而通过注册中心互相发现。

下面是我们的例子：

![]({{ site.url }}/images/springcloud/config_discovery.png)

1. 所有的微服务都会向配置中心（config server）读取配置文件。
2. 微服务都会注册到discovery server中。

下面我们按模块来说：

## config server （配置中心）
在pom.xml 中添加config server依赖
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
    <relativePath/>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
</dependencies>


<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
本文后面提到的pom，parent，dependencyManagement和这里是一样，只有dependencies这里有变化。后面只说dependencies的变化

ConfigServerApplication.java

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

在resources下面创建配置文件application.yml<br>

*Note: (springboot 默认配置文件名是application， 也可以在上面的mian中通过 System.setProperty("spring.config.name", "config-server") 重置spring.config.name为config-server,那么此时的会加载名为config-server.yml 或 config-server.properties文件)*
```yaml
spring:
  application:
    name: config-server
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
server:
  port: 1111 # 启动在这个端口
```
这里说明几个配置
- active: native
  <br>spring cloud config 有几种不同存储配置文件方式。git,svn,本地。这里的配置代表本地存储
- search-locations: classpath:/config
  <br>搜索路径为 resources下面的config目录。目录结构如下。
  ```
  resources/
    application.yml
    config/
      discovery-server.yml
      simple-service.yml
  ```
  config文件夹下面存放着 后面两个service的配置文件。文件内容我们后面再给出。


启动ConfigServerApplication, 在浏览器输入URL: http://localhost:1111/config/discovery-server
输出如下：
```
{"name":"config","profiles":["discovery-server"],"label":null,"version":null,"state":null,"propertySources":[]}
```
输入如下URL：http://localhost:1111/config/discovery-server.yml
接可以输出discovery-server.yml的文件内容

到这里，config server 准备完毕。

## netflix eureka (注册中心)
pom的依赖
```xml
<dependency>
    <!--注册中心的依赖-->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>

<dependency>
     <!--使该application成为config client  以便从config server获取配置-->
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```
DiscoveryServerApplication.java
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServerApplication.class, args);
    }
}
```
这个application既是配置中心的client端，又是注册中心的server端。

如何让我们的知道应用去哪里获取配置中心里的配置文件呢？
spring cloud 基于springboot 并在其application context上加了 父 context 也就是 bootstrap context， 也就是启动时会先加载bootstrap，然后才是application

因此我们可以在 bootstrap.yml 中定义config server的信息。如下:
```yaml
spring:
  application:
    name: discovery-server
  cloud:
    config:
      uri: http://localhost:1111/ # config server 的URL
```
这段配置使程序在启动时会去config server http://localhost:1111/ 去获取配置。再根据 config server中配置的 **search-locations: classpath:/config**下面去寻找 文件名为 **name: discovery-server**的文件

我们再来看 discovery-server.yml(文件在config server的config下面)的内容
```yaml
server:
  port: 2222

eureka:
  instance:
    hostname: localhost
  client:  # 不是一个注册中心的client，不需要注册自己。
    registerWithEureka: false
    fetchRegistry: false
```
这段配置描述了 discovery server启动在哪个端口


启动 DiscoveryServerApplication,在启动的log里可以看到下面的内容
```
Fetching config from server at : http://localhost:1111/
```
启动后之后。浏览器输入 http://localhost:2222/ 如下

![]({{ site.url }}/images/springcloud/spring-eureka-server.png)

我们从config server正确的获取了discovery-server.yml 并按照获取的配置内容 启动在了 2222 端口。上图可以看到 没有其他 service 注册到这里。我们继续往下


## 简单的micro service
我们要创建一个简单的web微服务，注册到注册中心，相关配置也是从配置中心获取。
pom.xml 里的依赖
```xml
<dependencies>
    <dependency>
        <!--让当前project成为 配置中心的客户端-->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>

    <dependency>
        <!--让当前project成为 注册中心的客户端-->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <!--web-->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```
创建SimpleServiceApplication.java
```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class SimpleServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SimpleServiceApplication.class,args);
    }

    @RequestMapping("/")
    public String home(){
        return "Hello";
    }
}
```
@EnableDiscoveryClient : 使该application成为 discovery的客户端（注册中心的客户端）。

在resources下面创建配置文件 bootstrap.yml，并配置config server的信息
```yaml
spring:
  application:
    name: simple-service
  cloud:
    config:
      uri: http://localhost:1111/
```
这段配置使程序启动时去config server获取 config 下面的名为 simple-service 的文件。我们来看下在第一步放入config server中 simple-service.yml 的内容
```yaml
server:
  port: 3333

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:2222/eureka/
```
这里的配置使simple-service 启动在 3333 端口。并注册到注册中心 http://localhost:2222/eureka/

启动SimpleServiceApplication,
在浏览器输入： http://localhost:3333/ 会出现 **Hello** 的字样。 程序从config server中获取了配置 并启动在了 3333 端口。

刷新下 http://localhost:2222/， 就能看到
![]({{ site.url }}/images/springcloud/simple-service-eureka.png)

service也注册到了注册中心中。

-------------------------------
至此我们通过简单的例子了解了 spring cloud config 和 spring cloud netflix eureka。这两个组件相当强大，提供的功能也远不止我这里描述的。 后面我们可以继续输入了解这些便捷的工具，以及工具背后的理论思想。
