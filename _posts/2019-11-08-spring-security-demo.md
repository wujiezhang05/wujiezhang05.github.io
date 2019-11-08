---
title: spring security + spring data jpa + thymeleaf 的login实例
category: Java
tags: spring
---

> spring security: 为企业应用程序提供认证，授权及其他安全功能
> spring data JPA(java presistence api) 数据持久化框架，
> thymeleaf: Java XML/HTML5/XHTML 模板引擎

我们用着三个组件完成一个小的login实例.

![]({{ site.url }}/images/springsecurity/spring-security.jpg)

## Overall

maven 依赖
```xml
<dependencies>
    <dependency>
        <!-- Setup Spring security support -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <dependency>
        <!-- Setup Spring thymeleaf support -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>

    <dependency>
        <!-- Setup Spring Data JPA Repository support -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <!-- Setup Spring web support -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- it brings useful tags to display spring security stuff -->
    <dependency>
        <groupId>org.thymeleaf.extras</groupId>
        <artifactId>thymeleaf-extras-springsecurity4</artifactId>
    </dependency>

    <!-- hot swapping, disable cache for template, enable live reload -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <!-- In-memory database for testing/demos -->
        <groupId>org.hsqldb</groupId>
        <artifactId>hsqldb</artifactId>
    </dependency>
</dependencies>
```
每个依赖的作用在注释里有标注，

spring boot 的application.yml

```yaml
spring:
  application:
    name: spring-security-demo
  freemarker:
    enabled: false
  thymeleaf:
    cache: false
    perfix: classpath:/templates/

server:
  port: 1111
```
thymeleaf 的HTML模板文件在resources/templates目录下

![]({{ site.url }}/images/springsecurity/spring-security-overall.PNG)


## GUI ---thymeleaf

login.html: 为了节省篇幅，一些外部资源文件CSS JS等的引入没有显示。
```HTML
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<form class="login100-form validate-form" method="post" th:action="@{/login}">
  <div class="wrap-input100 validate-input m-b-26" data-validate="Username is required">
    <span class="label-input100">Username</span>
    <input class="input100" type="text" name="username" placeholder="Enter username">
    <span class="focus-input100"></span>
  </div>

  <div class="wrap-input100 validate-input m-b-18" data-validate = "Password is required">
    <span class="label-input100">Password</span>
    <input class="input100" type="password" name="password" placeholder="Enter password">
    <span class="focus-input100"></span>
  </div>

  <!-- thymeleaf parameter, if authentication failed, this part will be shown. -->
  <div th:if="${param.error}">
    <span class="invalid-user">Invalid username and password.</span>
  </div>

  <div class="container-login100-form-btn">
    <button class="login100-form-btn" type="submit">
      Login
    </button>
  </div>
</form>
</html>
```
其他资源文件如 CSS JS等默然要放在resources/static下面, 如果需要在html中引入这些资源文件如下(启动时会去static目录下寻找 css/util.css)
```html
<link rel="stylesheet" type="text/css" href="css/util.css">
```
hello.html 登录成功之后显示的页面
```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="https://www.thymeleaf.org"
      xmlns:sec="https://www.thymeleaf.org/thymeleaf-extras-springsecurity3">
    <head>
        <title>Hello World!</title>
    </head>
    <body>
        <!--get the username from httpServletRequest after login sucess -->
        <h1 th:inline="text">Hello [[${#httpServletRequest.remoteUser}]]!</h1>
        <form th:action="@{/logout}" method="post">
            <input type="submit" value="Sign Out"/>
        </form>
    </body>
</html>
```


login的submit按钮会出发 th:action="@(/login)"，也即出发 login的post http request.


## Backend processor --spring boot security
我们来看project的入口类SecurityApplication
```Java
@SpringBootApplication
@Controller
public class SecurityApplication {

    public static void main(String[] args) {
        SpringApplication.run(SecurityApplication.class, args);
    }

    @RequestMapping(value = "/login", method = RequestMethod.GET)
    public String login() {
        return "login";
    }

    @RequestMapping("/hello")
    public String hello() {
        return "hello";
    }

    @RequestMapping("/")
    public String index() {
        return "login";
    }
}
```
上一篇关于 @Controller 和 @RequestMapping 已经介绍过了，这里的作用是
定义
- http://<IP>:<PORT>/login get请求  和 http://<IP>:<PORT>/  会访问 login.html,
- http://<IP>:<PORT>/hello 会访问 Hello.html
这里没有 login 的post请求的处理定义.

我们继续看配置security的类 WebSecurityConfig
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    @Autowired
    private DataSource dataSource;

    @Autowired
    private CustomUserDetailsService userDetailsService;


    @Override
    public void configure(final AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(encoder())
                .and()
                .authenticationProvider(authenticationProvider())
                .jdbcAuthentication()
                .dataSource(dataSource);
    }


    @Override
    public void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.authorizeRequests()
                .antMatchers("/", "/login").permitAll()
                .antMatchers("/hello").hasRole("USER")
                .and()
                .formLogin().loginPage("/login")
                .defaultSuccessUrl("/hello")
                .permitAll()
                .and()
                .logout().permitAll()
                .and().csrf().disable();
    }
    @Bean
    public PasswordEncoder encoder() {
        return new BCryptPasswordEncoder(11);
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        final DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(encoder());
        return authProvider;
    }
```
这里重点看连个configure 方法.
- configure(AuthenticationManagerBuilder auth)
  - 定义用JDBC的方式进行验证。
  - 校验逻辑在 CustomUserDetailsService
  - data source 在DataSource (在spirng data JPA会讲)
- configure(HttpSecurity httpSecurity)
  - 定义 http://<IP>:<PORT>/ 和 http://<IP>:<PORT>/login 任何用户都有权访问
  - 定义 http://<IP>:<PORT>/hello 的访问 必须是 USER 的ROLE 才可以访问,hasRole后面查询时会自动补全ROLE_, 所以数据库里数据角色是ROLE_USER 才有权限访问
  - 定义 login成功后默认跳转 http://<IP>:<PORT>/hello

我们来继续往下看 CustomUserDetailsService
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username);
        if (null == user) {
            throw new UsernameNotFoundException("user " + username + " is not found");
        }
        return new UserPrincipal(user);
    }
}
```
通过 userRepository 来访问后边的数据库获取用户的实例。这里只校验了用户是否存在。
我们继续看UserPrincipal
```java
public class UserPrincipal implements UserDetails {

    private final User user;

    UserPrincipal(User user) {
        this.user = user;
    }


    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER"));
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```
密码的校验framework做了，这里我们只定义了 如果校验成功 赋予改用户什么角色：ROLE_USER。

## dataRepository --spring data JPA
dataSource是在这里定义的。我们来看 UserConfiguration
```Java
@Configuration
@ComponentScan
@EntityScan("com.myself.study.database")
@EnableJpaRepositories("com.myself.study.database")
@PropertySource("classpath:db-config.properties")
public class UserConfiguration {
    @Bean
    public DataSource dataSource() {
        return  (new EmbeddedDatabaseBuilder()
                .addScript("classpath:/testdb/schema.sql")
                .addScript("classpath:/testdb/data.sql")
                .build());
    }
}
```
这里用的是嵌入式的数据库H2 数据库schema 数据在 resources/testdb下面
schema.sql
```sql
drop table users if exists;

create table users(
    username varchar(50) not null primary key,
    password varchar(100) not null,
    enabled boolean not null,
    );

drop table authorities if exists;

CREATE TABLE authorities(
  username varchar(50) NOT NULL,
  authority varchar(50) NOT NULL,
  FOREIGN KEY (username) REFERENCES users(username)
);

CREATE UNIQUE INDEX ix_auth_username on authorities (username,authority);
```

data.sql
```sql
insert into users (USERNAME, PASSWORD, ENABLED) values ('admin', '$2a$10$cbTrxgilMtq8VmLOHP8LneDG8FtobR.A.otxGmszbZOdNIFINkKiK', 'true');
insert into users (USERNAME, PASSWORD, ENABLED) values ('user', '$2a$10$e0OeINmS8viGvSNAVXaFsOwcqNGhhumEiw.Edm.QDaYM5CPVn7Aim', 'true');

INSERT INTO authorities (username, authority) values ('user', 'ROLE_USER');
INSERT INTO authorities (username, authority) values ('admin', 'ROLE_USER');
```
两个用户 user 和 admin，他们的role都是ROLE_USER， 密码是加密过的(参见WebSecurityConfig.java 里的BCryptPasswordEncoder)。
原始密码分别为 user，admin

db-config.properties
```properties
spring.jpa.hibernate.ddl-auto: validate
spring.jpa.hibernate.naming_strategy: org.hibernate.cfg.ImprovedNamingStrategy
spring.jpa.database: H2
spring.jpa.show-sql: true
```
UserDataRepository
```Java
public interface UserRepository extends Repository<User, String> {

    public User findByUsername(String username);
}
```
这里只定义了一个方法.

User.java
```java
@Entity
@Table(name = "users")
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    @Column(unique = true)
    @Id
    private String username;

    private String password;

    private boolean enabled;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

}
```
User对应的table users, UserDataRepository中findByUsername会去table users中获取数据并把数据映射为User的实例。


## 测试
启动SecurityApplication， 在浏览器中输入 http://localhost:1111/login

![]({{ site.url }}/images/springsecurity/spring-security-login.PNG)

密码错误：
![]({{ site.url }}/images/springsecurity/spring-security-login-error.PNG)

不login，直接浏览器访问 http://localhost:1111/hello,会发现被重定向到login页面。

密码正确：

![]({{ site.url }}/images/springsecurity/spring-security-login-suc.PNG)

再开一个标签页，访问 http://localhost:1111/hello, 发现可以访问。和上面内容一致。

--------------------------------------

这个例子中 所有内容都在同一个project里，在cloud环境里 需要各种不同微服务，GUI界面是独立microservice，
数据库也是。spring security也提供在cloud下的认证，授权方式。我们后面继续一起学习。