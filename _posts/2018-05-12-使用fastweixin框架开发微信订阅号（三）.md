---
layout: post
title: 使用fastweixin框架回复模板消息（三）。
---

本文使用fastweixin框架来群发消息，使用前请先阅读[使用fastweixin框架快速搭建微信公众平台的后台。](https://xinghelanchen.github.io/2018/05/11/%E4%BD%BF%E7%94%A8fastweixin%E6%A1%86%E6%9E%B6%E5%BC%80%E5%8F%91%E5%BE%AE%E4%BF%A1%E8%AE%A2%E9%98%85%E5%8F%B7.html)，本文使用了github上面sd4324530提供的[快速搭建微信公众平台服务器](https://github.com/sd4324530/fastweixin)。

## 前言
本文仅用于测试微信模板回复功能。

所以采用的main函数的方式

开始前请先登陆微信工作平台[测试号管理](https://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)，获取你的appID和appsecret



### 回复模板消息

请先在[测试号管理](https://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)中增加一个模板，记录模板的id和在模板内容中填入“注入成功：{{succ.DATA}} 注入失败：{{err.DATA}}”类似的内容，注入数据的格式必须按照微信的要求。

修改第一篇博文中的WeixinController.java

```
import com.github.sd4324530.fastweixin.api.TemplateMsgAPI;
import com.github.sd4324530.fastweixin.api.config.ApiConfig;
import com.github.sd4324530.fastweixin.api.entity.TemplateMsg;
import com.github.sd4324530.fastweixin.api.entity.TemplateParam;
import com.github.sd4324530.fastweixin.message.BaseMsg;
import com.github.sd4324530.fastweixin.message.TextMsg;
import com.github.sd4324530.fastweixin.message.req.TextReqMsg;
import com.github.sd4324530.fastweixin.servlet.WeixinControllerSupport;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

import java.util.HashMap;
import java.util.Map;

@RequestMapping("/wxtest/hello")
@Controller
public class WeixinController extends WeixinControllerSupport {
    private static final Logger log = LoggerFactory.getLogger(WeixinController.class);
    private static final String TOKEN = "yourtoken";
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
        if ("test".equals(content)) {
            String appID = "wx****************";//这里填写你自己的appID
            String appsecret = "********************************";//这里填写你自己的appsecret
            ApiConfig config = new ApiConfig(appID, appsecret);

            TemplateMsgAPI templateMsgAPI = new TemplateMsgAPI(config);

            TemplateMsg templateMsg = new TemplateMsg();
            templateMsg.setTouser(msg.getFromUserName());
            templateMsg.setTemplateId("ix*****************************************");//填入你自己的模板id
            Map<String, TemplateParam> map = new HashMap<>();
            map.put("succ", new TemplateParam("202"));//这里就是用来创建回复消息里面的数据
            map.put("err", new TemplateParam("11"));
            templateMsg.setData(map);
            templateMsgAPI.send(templateMsg);
        }
    }
}

```
### 测试
手机给订阅号发送test，查看是不是可以收到设定好的模板消息和正确的数据。


转载请标注原文链接
