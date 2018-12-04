---
layout: post
title: springboot 集成 spring security
categories: springboot
description: springboot 集成 spring security
keywords: Java, springboot, spring security
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
//登录错误页面
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

启动以后访问 http://127.0.0.1:8080/hello ，会自动跳转到 http://127.0.0.1:8080/login 

现在打开的就是自己的登录页面了

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532946453.png)

输入错误的用户名和密码

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532946593.png)

输入正确的用户名和密码

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1532946688.png)

就可以跳转到之前要访问的链接了。

当然我们的目标并不是这样

## 3. 自定义用户名和密码

官网的demo我就不贴了，这里直接写从数据库查用户名和密码的方式。

spring security的原理就是使用很多的拦截器对URL进行拦截，以此来管理登录验证和用户权限验证。

用户登陆，会被AuthenticationProcessingFilter拦截，调用AuthenticationManager的实现，而且AuthenticationManager会调用ProviderManager来获取用户验证信息（不同的Provider调用的服务不同，因为这些信息可以是在数据库上，可以是在LDAP服务器上，可以是xml配置文件上等），如果验证通过后会将用户的权限信息封装一个User放到spring的全局缓存SecurityContextHolder中，以备后面访问资源时使用。

所以我们要自定义用户的校验机制的话，我们只要实现自己的AuthenticationProvider就可以了。在用AuthenticationProvider 这个之前，我们需要提供一个获取用户信息的服务，实现  UserDetailsService 接口

用户名密码->(Authentication(未认证)  ->  AuthenticationManager ->AuthenticationProvider->UserDetailService->UserDetails->Authentication(已认证）

### 3.1 定义自己的用户类继承 UserDetails 和 Serializable 接口

```
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.AuthorityUtils;
import org.springframework.security.core.userdetails.UserDetails;

import java.io.Serializable;
import java.util.Collection;

public class UserInfo implements Serializable, UserDetails {

    private static final long serialVersionUID = 1L;
    private String username;
    private String password;
    private String role;
    private boolean accountNonExpired;
    private boolean accountNonLocked;
    private boolean credentialsNonExpired;
    private boolean enabled;
    public UserInfo(String username, String password, String role, boolean accountNonExpired, boolean accountNonLocked,
                    boolean credentialsNonExpired, boolean enabled) {
        // TODO Auto-generated constructor stub
        this.username = username;
        this.password = password;
        this.role = role;
        this.accountNonExpired = accountNonExpired;
        this.accountNonLocked = accountNonLocked;
        this.credentialsNonExpired = credentialsNonExpired;
        this.enabled = enabled;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return AuthorityUtils.commaSeparatedStringToAuthorityList(role);
    }
    get... \ set...
}
```
### 3.2 MyUserDetailsService 用来返回UserInfo实例
```
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Component;

@Component
public class MyUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        //这里应该去数据库查询构建userInfo，此处模拟一下
        if (username.equals("admin")) {
            UserInfo userInfo = new UserInfo("admin", "123456", "ROLE_ADMIN", true, true, true, true);
            return userInfo;
        }

        return null;
    }
}
```
### 3.3 实现自己的 MyAuthenticationProvider 这个里面就是用来自己做登录校验了
```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.stereotype.Component;

import java.util.Collection;

@Component
public class MyAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName(); //用户输入的密码
        String password = (String) authentication.getCredentials(); //用户输入的密码

        System.out.println(username + password);

        //开始校验用户
        UserInfo userInfo = (UserInfo) userDetailsService.loadUserByUsername(username);

        if (userInfo == null) {
            throw new BadCredentialsException("用户名不存在");
        }

        // //这里我们还要判断密码是否正确，实际应用中，我们的密码一般都会加密，以Md5加密为例
        // Md5PasswordEncoder md5PasswordEncoder=new Md5PasswordEncoder();
        // //这里第个参数，是salt
        // 就是加点盐的意思，这样的好处就是用户的密码如果都是123456，由于盐的不同，密码也是不一样的，就不用怕相同密码泄漏之后，不会批量被破解。
        // String encodePwd=md5PasswordEncoder.encodePassword(password, userName);
        // //这里判断密码正确与否
        // if(!userInfo.getPassword().equals(encodePwd))
        // {
        // throw new BadCredentialsException("密码不正确");
        // }
        // //这里还可以加一些其他信息的判断，比如用户账号已停用等判断，这里为了方便我接下去的判断，我就不用加密了。

        if (!userInfo.getPassword().equals(password)) {
            throw new BadCredentialsException("密码不正确");
        }

        Collection<? extends GrantedAuthority> authorities = userInfo.getAuthorities();

        // 构建返回的用户登录成功的token
        return new UsernamePasswordAuthenticationToken(userInfo, password, authorities);
    }

    @Override
    public boolean supports(Class<?> aClass) {
        //这里改成true，才算是开启了。
        return true;
    }
}
```
现在校验部分完成了，我们需要在 SecurityConfig 中配置我们自己的校验

```
@Autowired
private AuthenticationProvider provider;

@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.authenticationProvider(provider);
}
```

现在重新运行程序，需要输入的用户名 admin ，密码 123456 ，就可以访问了。

为了方便测试，可以添加一个 controller 用来查看当前登录的用户信息。
```
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class WhoController {

    @RequestMapping("/whoim")
    @ResponseBody
    public Object whoIm() {
        return SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    }
}
```
这样登录以后就可以访问 http://127.0.0.1:8080/whoim 来查看当前登录的用户的信息了

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1533025115.png)

到这里自定义登录已经成功了。

## 4. 自定义登录成功和失败的处理逻辑

为了实现登录成功或者失败以后可以跳转到指定页面或者返回json格式的数据，这里需要新建两个类

```
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.web.DefaultRedirectStrategy;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;
import org.springframework.stereotype.Component;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@Component("myAuthenticationSuccessHandler")
public class MyAuthenticationSuccessHandler extends SavedRequestAwareAuthenticationSuccessHandler {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws ServletException, IOException {
//        super.onAuthenticationSuccess(request, response, authentication);
        // 跳转到指定页面
        new DefaultRedirectStrategy().sendRedirect(request, response, "/hello");
    }
}
```

```
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.authentication.SimpleUrlAuthenticationFailureHandler;
import org.springframework.stereotype.Component;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

@Component("myAuthenticationFailureHandler")
public class MyAuthenticationFailHander extends SimpleUrlAuthenticationFailureHandler {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
//        super.onAuthenticationFailure(request, response, exception);
        //以Json格式返回
        Map<String,String> map = new HashMap<>();
        map.put("code", "201");
        map.put("msg", "登录失败");
        response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(objectMapper.writeValueAsString(map));
    }
}
```

增加这两个以后，还需要修改 SecurityConfig 类进行配置，修改 configure 方法

```
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .formLogin().loginPage("/login").loginProcessingUrl("/login/form")
                .successHandler(myAuthenticationSuccessHandler)
                .failureHandler(myAuthenticationFailureHandler)
                .permitAll()
                .and()
                .authorizeRequests().anyRequest().authenticated()
                .and()
                .csrf().disable();
    }
```

## 5. 添加基于 RBAC(role-Based-access control) 权限控制

权限控制一般都是由三个部分组成，用户、角色、资源（菜单、按钮），以及用户和角色的关联表，角色和资源的关联表

核心是判断用户要访问的url是否存在于用户角色所拥有的权限列表中

```
RbacService 接口类
import org.springframework.security.core.Authentication;

import javax.servlet.http.HttpServletRequest;

public interface RbacService {
    boolean hasPermission(HttpServletRequest request, Authentication authentication);
}
```
```
RbacService 实现类
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;
import org.springframework.util.AntPathMatcher;

import javax.servlet.http.HttpServletRequest;
import java.util.HashSet;
import java.util.Set;

@Service("rbacService")
public class RbacServiceImpl implements RbacService {
    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public boolean hasPermission(HttpServletRequest request, Authentication authentication) {

        Object principal = authentication.getPrincipal();
        boolean hasPermission = false;
        if (principal instanceof UserDetails) {
            String userName = ((UserDetails) principal).getUsername();
            Set<String> urls = new HashSet<>(); //数据库读取该用户的角色拥有的权限

            urls.add("/hello");
            // 这里面判断 url 不能用 equals() ，因为有的 url 是有参数的。
            for (String url : urls) {
                if (antPathMatcher.match(url, request.getRequestURI())) {
                    hasPermission = true;
                    break;
                }
            }
        }
        return hasPermission;
    }
}
```
然后修改 SecurityConfig 类中的验证方式

```
http
        .formLogin().loginPage("/login").loginProcessingUrl("/login/form")
        .successHandler(myAuthenticationSuccessHandler)
        .failureHandler(myAuthenticationFailureHandler)
        .permitAll()
        .and()
        .authorizeRequests()
        .anyRequest().access("@rbacService.hasPermission(request,authentication )")
        .and()
        .csrf().disable();
```

现在访问 /hello ，以后正确登录就可以打开 hello ，但是访问 /whoim 的时候就会报403，没有权限

## 6. 记住我功能 Remeber me

这里我选用了使用数据库存储token，这样后台重启以后， Remeber me 依然生效，

### 6.1 先创建表
```
CREATE TABLE persistent_logins (
    username VARCHAR(64) NOT NULL,
    series VARCHAR(64) NOT NULL,
    token VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL,
    PRIMARY KEY (series)
);
```
### 6.2 页面增加勾选
```
<tr>
    <td>记住我</td>
    <td><input type="checkbox" name="remember-me"  class="form-control"/></td>
</tr>
```
### 6.3 增加依赖包
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```
### 6.4 SecurityConfig 类里面配置好 token 的存储及数据源
```
@Autowired
    DataSource dataSource;
    /**
     * 记住我功能的token存取器配置
     * @return
     */
    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
        tokenRepository.setDataSource(dataSource);
        return tokenRepository;
    }
```
### 6.5 修改 SecurityConfig 中的 configure(HttpSecurity http) 方法
```
http
        .formLogin().loginPage("/login").loginProcessingUrl("/login/form")
        .successHandler(myAuthenticationSuccessHandler)
        .failureHandler(myAuthenticationFailureHandler)
        .permitAll()
        .and()
        .rememberMe()
            .rememberMeParameter("remember-me").userDetailsService(userDetailsService)
            .tokenRepository(persistentTokenRepository())
            .tokenValiditySeconds(3600 * 24 * 7) // 配置token有效期
        .and()
        .authorizeRequests()
        .anyRequest().access("@rbacService.hasPermission(request,authentication )")
        .and()
        .csrf().disable();
```
现在登录以后，重启后台程序，然后可以直接访问 /hello 页面，不需要登录。

登录以后数据库会有该用户的token信息和最后时间

![Image text](https://raw.githubusercontent.com/xinghelanchen/xinghelanchen.github.io/master/_img/1533550600.png)

## 本文参考了[springboot 集成 spring security](https://blog.csdn.net/qq_29580525/article/details/79317969)

转载请标注原文链接
