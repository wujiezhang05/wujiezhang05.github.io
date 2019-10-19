---
title: Spring boot的学习和思考
excerpt: 微服务时代，springboot以约定大于配置的设计理念，简化了之前臃肿的配置方式，大大提高饿了开发效率.让我们通过一个例子来学习分析下
category: Java
last_modified_at: 2017-03-09T10:27:01-05:00
---

> 微服务时代，springboot以约定大于配置的设计理念，简化了之前臃肿的配置方式，大大提高饿了开发效率
> 让我们通过一个例子来学习下

---------------------

## 创建Applicaton
在pom.xml 中加入下面配置
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>1.5.12.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

```
**为什么要把spring-boot-starter-parent作为我们project的parent呢？**

  因为这个project是spring提供一个特殊的project，在里面提供了很多默认配置（如：定义java编译版本，定义UTF-8编码格式，提供spring-boot-dependencies等）， 设置这个这个project，我们就可以继承这些配置。不需要我们重复配置了,如：上面我们的依赖就没有声明版本。

*note: 如果你需要你自己的parent，也可以通过dependencyManagement来自己设置相关的配置。这里不延伸*

创建application class

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @RequestMapping("/index")
    public String home(){
        return "Hello World";
    }
}
```
我们来分析
- @SpringBootApplication
  是几个注解的集合
    - @Configuration <br/>
      表示class 定义了 Bean，需要注册到spring 容器中
    - @EnableAutoConfiguration <br/>
    表示springboot会自动根据classpath，properties等配置来自动添加bean，例如：我们的例子中加入了
    spring-boot-starter-web. *spring-webmvc*在我们的classpath里,这个application就会被认为是一个web工程并初始化一些相关的步骤。
    - @ComponentScan <br/>
    告诉springboot当前package下扫描其他的组件：如Controller等

- SpringApplication.run(Application.class, args) <br/>
  启动一个web applicaton，没有web.xml, springboot封装了tomcat。

- @RestController 和 @RequestMapping <br/>
  捕获并处理REST请求<br/>
    - @RestController 是 @Controller 和 @ResponseBody的集合。<br/>
      @Controller 捕获REST请求，@ResponseBody 返回的String 作为 responsebody
    - @RequestMapping <br/>
      处理http请求，如: http://localhost:8080/index 就会到 被标注的 方法


启动applicaton，在浏览器中输入URL， 如图<br/>
![]({{ site.url }}/images/springboot/url.PNG)

## 单元测试

我们在来写一个单元测试。 在pom.xml 中加入
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```
写一个简单的单元测试
```java
import org.hamcrest.Matchers;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class ApplicationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void getHome() throws Exception{
        mockMvc.perform(MockMvcRequestBuilders.get("/index").accept(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().string(Matchers.equalTo("Hello World")));
    }
}
```
继续分析
@SpringBootTest
  告诉springboot寻找一个主配置类（@SpringBootApplication），并用它来启动spring，加载完整的应用程序（在我们的例子中单元测试会真的建立http连接）。
@RunWith(SpringRunner.class)
  用来提供spring boot test 和 junit 功能之间的桥梁

## 总结
springboot 是一个很好的工具帮助我们快速构建我们的项目，更快更简单也是编程领域的趋势。或许有一天一些简单的编程工作真的可以被替代了。我们在学习工具使用的同时，也要了解下他们设计的原理。知其然并知其所以然。
