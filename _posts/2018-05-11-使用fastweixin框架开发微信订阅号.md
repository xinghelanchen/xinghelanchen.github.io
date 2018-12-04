---
layout: post
title: 使用fastweixin框架快速搭建微信公众平台的后台
categories: 微信后台
description: 使用fastweixin框架快速搭建微信公众平台的后台
keywords: Java, 微信公众号
---

本文用于介绍在springboot项目中使用fastweixin框架，快速搭建微信公众平台的后台。使用了github上面sd4324530提供的[快速搭建微信公众平台服务器](https://github.com/sd4324530/fastweixin)。

## 前言
本文用于介绍在springboot项目中使用fastweixin框架，快速搭建微信公众平台的后台。

阅读本文之前请先熟悉springboot工程的创建，官网教程：https://projects.spring.io/spring-boot/#quick-start

开发环境：IDEA、JDK1.8
## 正文
微信公众平台接口测试帐号申请：https://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login

登陆以后需要配置“接口配置信息”

这里的Token可以自行填写，这里的URL需要写一个公网可以访问的接口地址，到这里就需要在本地新建springboot工程了。

## 操作如下

### 在IDEA中新建一个springboot工程
pom.xml中增加如下依赖：

```
<dependency>
    <groupId>com.github.sd4324530</groupId>
    <artifactId>fastweixin</artifactId>
    <version>1.3.15</version>
</dependency>
```

### 在controller下增加如下的一个类

```
import com.github.sd4324530.fastweixin.message.BaseMsg;
import com.github.sd4324530.fastweixin.message.TextMsg;
import com.github.sd4324530.fastweixin.message.req.TextReqMsg;
import com.github.sd4324530.fastweixin.servlet.WeixinControllerSupport;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@RequestMapping("/wxtest/hello")
@Controller
public class WeixinController extends WeixinControllerSupport {
    private static final Logger log = LoggerFactory.getLogger(WeixinController.class);
    private static final String TOKEN = "yourtoken";//这里填写你的token
    //设置TOKEN，用于绑定微信服务器
    @Override
    protected String getToken() {
        return TOKEN;
    }

    //使用安全模式时设置：APPID
    //不再强制重写，有加密需要时自行重写该方法
    @Override
    protected String getAppId() {
        return null;
    }

    //使用安全模式时设置：密钥
    //不再强制重写，有加密需要时自行重写该方法
    @Override
    protected String getAESKey() {
        return null;
    }

    //重写父类方法，处理对应的微信消息
    @Override
    protected BaseMsg handleTextMsg(TextReqMsg msg) {
        String content = msg.getContent();
        log.info("用户发送到服务器的内容:{}", content);
        return new TextMsg("服务器回复用户消息!");
    }
}
```

### 启动springboot工程
1. 确认上述的http://localhost:8080/wxtest/hello在本地可以访问
2. cmd→ipconfig 查看本地的ip，把本地的http://ip:8080/wxtest/hello映射为外网的http://外网ip:80/wxtest/hello
3. 在“接口配置信息”里面的URL填写外网的链接。
4. 点击“提交”按钮，正常应该是显示“配置成功”

### 测试
手机扫下面的“测试号二维码”，以后发送消息，应该可以在控制台打印发送的消息，并且在手机上收到服务器回复的消息。

转载请标注原文链接
