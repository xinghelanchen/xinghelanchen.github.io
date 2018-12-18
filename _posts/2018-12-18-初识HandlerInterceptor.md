---
layout: post
title: 使用拦截器让页面可以获取basePath
categories: JAVA_WEB
description: 使用拦截器让页面可以获取basePath
keywords: Java, SpringBoot
---

使用 SpringBoot + Freemarker 的时候，页面要使用 basePath 。百度找了几个方案，这里先说拦截器的方案。

### 1. 先建立公共的拦截器 CommonInterceptor 实现 HandlerInterceptor 接口

```
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author yantao
 * 公共拦截器
 */
public class CommonInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String path = request.getContextPath();
        String scheme = request.getScheme();
        String serverName = request.getServerName();
        int port = request.getServerPort();
        String basePath = scheme + "://" + serverName + ":" + port + path;
        request.setAttribute("basePath", basePath);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
```

### 2. 然后新建 CommonInterceptorConfig 实现 WebMvcConfigurer 接口

```
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * @author yantao
 * 注册拦截器
 */
@Configuration
public class CommonInterceptorConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new CommonInterceptor()).addPathPatterns("/**");
    }
}
```

> 在 SpringBoot 2.0 以后要用 WebMvcConfigurer 接口，如果继承 WebMvcConfigurationSupport 的话，自动配置的静态资源路径（classpath:/META/resources/，classpath:/resources/，classpath:/static/，classpath:/public/）就不生效了。具体参看下面的源码：

```
package org.springframework.boot.autoconfigure.web.servlet;

import ...

@Configuration
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {
    ...
}
```

```
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
```

这个注解的意思是在项目类路径中 缺少 WebMvcConfigurationSupport类型的bean时改自动配置类才会生效，所以继承 WebMvcConfigurationSupport 后需要自己再重写相应的方法。

### 3. 现在就可以在 .ftl 文件中直接使用 ${basePath} 了。

```
var serverWeb = "${basePath}";
```

