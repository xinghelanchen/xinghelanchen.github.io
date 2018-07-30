---
layout: post
title: springboot 集成 spring security。
---

工作需要在 springboot 中集成 spring security，所以网上研究了一下

## 1. 添加 spring security 依赖

```
<!-- spring security依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

启动工程，访问任意一个 url 都会跳转到登录页面

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532924392.png)

用户名是 user ，密码是启动时候在控制台打印出来的

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532925340.png)

## 2. 开始改造登陆页面

先新建一个类重写 WebSecurityConfigurerAdapter 的 configure(HttpSecurity http) 方法

```
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .formLogin().loginPage("/login").loginProcessingUrl("/login/form").failureForwardUrl("/login-error").permitAll()
                .and()
                .authorizeRequests().anyRequest().authenticated()
                .and()
                .csrf().disable();
    }
}
```

在上面这个类里面 /login 是跳转到登录页面的 url ， /login/form 是登录页面的表单提交的 url ， /login-error 是前两个页面访问失败要跳转的页面，因为这三个页面是不需要登录就可以访问的，所以要使用 .permitAll() 标注。

```
//正常的要访问的页面
@Controller
public class HelloController {
	
    @RequestMapping("/hello")
    @ResponseBody
    public String index() {
        return "Hello Spring Boot!";
    }
}
```

```
//登陆控制器
@Controller
public class LoginController {

    @RequestMapping("/login")
    public String userLogin() {
        return "login";
    }

    @RequestMapping("/login-error")
    public String loginError() {
        return "login-error";
    }
}
```

```
//登录页面
<html>
<head>
    <title>登录</title>
</head>
<body>
    <form  class="form-signin" action="/login/form" method="post">
        <h2 class="form-signin-heading">用户登录</h2>
        <table>
            <tr>
                <td>用户名:</td>
                <td><input type="text" name="username"  class="form-control"  placeholder="请输入用户名"/></td>
            </tr>
            <tr>
                <td>密码:</td>
                <td><input type="password" name="password"  class="form-control" placeholder="请输入密码" /></td>
            </tr>
            <tr>
                <td colspan="2">
                    <button type="submit"  class="btn btn-lg btn-primary btn-block" >登录</button>
                </td>
            </tr>
        </table>
    </form>
</body>
</html>
```

```
<!DOCTYPE HTML>
<html>
<head>
    <title>用户登录</title>
</head>
<body>
    <h3>用户名或密码错误</h3>
</body>
</html>
```

转载请标注原文链接
