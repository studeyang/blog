---
permalink: 2023/0912.html
title: Spring Security入门：保护Web应用
date: 2023-09-12 09:00:00
tags: Spring Security
cover: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202309112237318.jpg
thumbnail: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202309112237318.jpg
categories: technotes
toc: true
description: 本文我们将构建一个简单但完整的小型 Web 应用程序，以演示 Spring Security 的入门教程。系统大致逻辑是：当合法用户成功登录系统之后，浏览器会跳转到一个系统主页，并展示一些个人健康档案（HealthRecord）数据。
---

本文我们将构建一个简单但完整的小型 Web 应用程序，以演示 Spring Security 的入门教程。大致逻辑是：当合法用户成功登录系统之后，浏览器会跳转到一个系统主页，并展示一些个人健康档案（HealthRecord）数据。

让我们开始吧！

## 系统初始化

这部分工作涉及**领域对象的定义、数据库初始化脚本的整理以及相关依赖组件的引入**。

针对领域对象，我们重点来看如下所示的 User 类定义：

```java
@Entity
public class User {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
 
    private String username;
    private String password;
 
    @Enumerated(EnumType.STRING)
    private PasswordEncoderType passwordEncoderType;
 
    @OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
	private List<Authority> authorities;
	…
}
```

```java
public enum PasswordEncoderType {
    BCRYPT, SCRYPT
}
```

可以看到，这里除了指定主键 id、用户名 username 和密码 password 之外，还包含了一个**加密算法枚举值 EncryptionAlgorithm**。在案例系统中，我们将提供 BCryptPasswordEncoder 和 SCryptPasswordEncoder 这两种可用的密码解密器，你可以通过该枚举值进行设置。

同时，我们在 User 类中还发现了一个 Authority 列表。显然，这个列表用来指定该 User 所具备的权限信息。Authority 类的定义如下所示：

```java
@Entity
public class Authority {
 
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
 
    private String name;
 
    @JoinColumn(name = "user")
    @ManyToOne
	private User user;
	…
}
```

通过定义不难看出 User 和 Authority 之间是**一对多**的关系。

基于 User 和 Authority 领域对象，我们也给出创建数据库表的 SQL 定义，如下所示：

```sql
CREATE TABLE IF NOT EXISTS `spring_security`.`user` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(45) NOT NULL,
  `password` TEXT NOT NULL,
  `password_encoder_type` VARCHAR(45) NOT NULL,
  PRIMARY KEY (`id`));

CREATE TABLE IF NOT EXISTS `spring_security`.`authority` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(45) NOT NULL,
  `user` INT NOT NULL,
  PRIMARY KEY (`id`));
  
CREATE TABLE IF NOT EXISTS `spring_security`.`health_record` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `username` VARCHAR(45) NOT NULL,
  `name` VARCHAR(45) NOT NULL,
  `value` VARCHAR(45) NOT NULL,
  PRIMARY KEY (`id`));
```

在运行系统之前，我们同样也需要初始化数据，对应脚本如下所示：

```sql
INSERT IGNORE INTO `spring_security`.`user` (`id`, `username`, `password`, `password_encoder_type`) VALUES ('1', 'studeyang', '$2a$10$xn3LI/AjqicFYZFruSwve.681477XaVNaUQbr1gioaWPn4t1KsnmG', 'BCRYPT');

INSERT IGNORE INTO `spring_security`.`authority` (`id`, `name`, `user`) VALUES ('1', 'READ', '1');
INSERT IGNORE INTO `spring_security`.`authority` (`id`, `name`, `user`) VALUES ('2', 'WRITE', '1');

INSERT IGNORE INTO `spring_security`.`health_record` (`id`, `username`, `name`, `value`) VALUES ('1', 'studeyang', 'weight', '70');
INSERT IGNORE INTO `spring_security`.`health_record` (`id`, `username`, `name`, `value`) VALUES ('2', 'studeyang', 'height', '177');
INSERT IGNORE INTO `spring_security`.`health_record` (`id`, `username`, `name`, `value`) VALUES ('3', 'studeyang', 'bloodpressure', '70');
INSERT IGNORE INTO `spring_security`.`health_record` (`id`, `username`, `name`, `value`) VALUES ('4', 'studeyang', 'pulse', '80');
```

这里初始化了一个用户名为 “studeyang”的用户，同时指定了它的密码为“12345”，加密算法为“BCRYPT”。

现在，领域对象和数据层面的初始化工作已经完成了，接下来我们需要在代码工程的 pom 文件中添加如下所示的 Maven 依赖：

```xml
<dependencies>	    
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
 
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
</dependencies>
```

## 实现用户管理

实现自定义用户认证的过程通常涉及两大部分内容，一方面需要使用 User 和 Authority 对象来完成**定制化的用户管理**，另一方面需要把这个定制化的用户管理**嵌入整个用户认证流程中**。

如果你想实现自定义的用户信息，扩展 UserDetails 这个接口即可。实现方式如下所示：

```java
public class CustomUserDetails implements UserDetails {
 
    private final User user;
 
    public CustomUserDetails(User user) {
        this.user = user;
    }
 
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getAuthorities().stream()
                   .map(a -> new SimpleGrantedAuthority(a.getName()))
                   .collect(Collectors.toList());
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
 
    public final User getUser() {
        return user;
    }
}
```

请注意，这里的 getAuthorities() 方法中，我们将 User 对象中的 Authority 列表转换为了 Spring Security 中代表用户权限的 **SimpleGrantedAuthority 列表**。

所有的自定义用户信息和权限信息都是维护在数据库中的，所以为了获取这些信息，我们需要创建数据访问层组件，这个组件就是 UserRepository，定义如下：

```java
public interface UserRepository extends JpaRepository<User, Integer> {
 
    Optional<User> findUserByUsername(String username);
}
```

现在，我们已经能够在数据库中维护自定义用户信息，也能够根据这些用户信息获取到 UserDetails 对象，那么接下来要做的事情就是扩展 UserDetailsService。自定义 CustomUserDetailsService 实现如下所示：

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {
 
    @Autowired
    private UserRepository userRepository;
 
    @Override
    public CustomUserDetails loadUserByUsername(String username) {
        Supplier<UsernameNotFoundException> s =
                () -> new UsernameNotFoundException("Username" + username + "is invalid!");
 
        User u = userRepository.findUserByUsername(username).orElseThrow(s);
 
        return new CustomUserDetails(u);
    }
}
```

这里我们通过 UserRepository 查询数据库来获取 CustomUserDetails 信息。

## 实现认证流程

实现自定义认证流程要做的也是实现 AuthenticationProvider 中的这两个方法，而认证过程势必要借助于前面介绍的 CustomUserDetailsService。

我们先来看一下 AuthenticationProvider 接口的实现类 AuthenticationProviderService，如下所示：

```java
@Service
public class AuthenticationProviderService implements AuthenticationProvider {
 
    @Autowired
    private CustomUserDetailsService userDetailsService;
 
    @Autowired
    private BCryptPasswordEncoder bCryptPasswordEncoder;
 
    @Autowired
    private SCryptPasswordEncoder sCryptPasswordEncoder;
 
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();
 
        //根据用户名从数据库中获取 CustomUserDetails
        CustomUserDetails user = userDetailsService.loadUserByUsername(username);
 
        //根据所配置的密码加密算法分别验证用户密码
        switch (user.getUser().getPasswordEncoderType()) {
            case BCRYPT:
                return checkPassword(user, password, bCryptPasswordEncoder);
            case SCRYPT:
                return checkPassword(user, password, sCryptPasswordEncoder);
        }
 
        throw new  BadCredentialsException("Bad credentials");
    }
 
    @Override
    public boolean supports(Class<?> aClass) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(aClass);
    }
 
    private Authentication checkPassword(CustomUserDetails user, String rawPassword, PasswordEncoder encoder) {
        if (encoder.matches(rawPassword, user.getPassword())) {
            return new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword(), user.getAuthorities());
        } else {
            throw new BadCredentialsException("Bad credentials");
        }
    }
}
```

我们首先通过 CustomUserDetailsService 从数据库中获取用户信息并构造成 CustomUserDetails 对象。然后，根据指定的密码加密器对用户密码进行验证，如果验证通过则构建一个 UsernamePasswordAuthenticationToken 对象并返回，反之直接抛出 BadCredentialsException 异常。而在 supports() 方法中指定的就是这个目标 UsernamePasswordAuthenticationToken 对象。

## 安全配置

最后，我们要做的就是通过 Spring Security 提供的配置体系将前面介绍的所有内容串联起来，如下所示：

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
 
    @Autowired
    private AuthenticationProviderService authenticationProvider;
 
    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }
 
    @Bean
    public SCryptPasswordEncoder sCryptPasswordEncoder() {
        return new SCryptPasswordEncoder();
    }
 
    @Override
    protected void configure(AuthenticationManagerBuilder auth) {
        auth.authenticationProvider(authenticationProvider);
    }
 
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
            .defaultSuccessUrl("/healthrecord", true);
        http.authorizeRequests().anyRequest().authenticated();
    }
}
```

这里注入了已经构建完成的 AuthenticationProviderService，并初始化了两个密码加密器 BCryptPasswordEncoder 和 SCryptPasswordEncoder。最后，我们覆写了 WebSecurityConfigurerAdapter 配置适配器类中的 configure() 方法，并指定用户登录成功后将跳转到"/healthrecord"路径所指定的页面。

对应的，我们需要构建如下所示的 HealthRecordController 类来指定"/healthrecord"路径，并展示业务数据的获取过程，如下所示：

```java
@Controller
public class HealthRecordController {
    @Autowired
    private HealthRecordService healthRecordService;
    
    @GetMapping("/main")
    public String main(Authentication a, Model model) {
    	String userName = a.getName();
        model.addAttribute("username", userName);
        model.addAttribute("healthRecords", healthRecordService.getHealthRecordsByUsername(userName));
        return "main.html";
    }
}
```

这里所指定的 health_record.html 位于 resources/templates 目录下，该页面基于 thymeleaf 模板引擎构建，如下所示：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>健康档案</title>
    </head>
    <body>
        <h2 th:text="'登录用户：' + ${username}" />
        <p><a href="/logout">退出登录</a></p>
        <h2>个人健康档案:</h2>
        <table>
            <thead>
            <tr>
                <th> 健康指标名称 </th>
                <th> 健康指标值 </th>
            </tr>
            </thead>
            <tbody>
            <tr th:if="${healthRecords.empty}">
                <td colspan="2"> 无健康指标 </td>
            </tr>
            <tr th:each="healthRecord : ${healthRecords}">
                <td><span th:text="${healthRecord.name}"> 健康指标名称 </span></td>
                <td><span th:text="${healthRecord.value}"> 健康指标值 </span></td>
            </tr>
            </tbody>
        </table>
    </body>
</html>
```

这里我们从 Model 对象中获取了认证用户信息以及健康档案信息，并渲染在页面上。

## 案例演示

现在，让我们启动 Spring Boot 应用程序，并访问[http://localhost:8080](http://localhost:8080/?fileGuid=xxQTRXtVcqtHK6j8)端点。因为访问系统的任何端点都需要认证，所以 Spring Security 会自动跳转到如下所示的登录界面：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023image-20230911113950836.png)

我们分别输入用户名“studeyang”和密码“12345”，系统就会跳转到健康档案主页：

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023image-20230911114211416.png)

在这个主页中，我们正确获取了登录用户的用户名，并展示了个人健康档案信息。这个结果也证实了自定义用户认证体系的正确性。

## 小结

本文实现了自定义的用户认证流程，作为 Spring Security 的入门示例，适合初学者进行学习，源码分享在：https://github.com/studeyang/spring-security-example

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202309112237318.jpg)
