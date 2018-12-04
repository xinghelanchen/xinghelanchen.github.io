---
layout: post
title: 使用fastweixin框架群发消息（二）
categories: 微信后台
description: 使用fastweixin框架群发消息（二）
keywords: Java, 微信公众号
---

本文使用fastweixin框架来群发消息，使用前请先阅读[使用fastweixin框架快速搭建微信公众平台的后台。](https://xinghelanchen.github.io/2018/05/11/%E4%BD%BF%E7%94%A8fastweixin%E6%A1%86%E6%9E%B6%E5%BC%80%E5%8F%91%E5%BE%AE%E4%BF%A1%E8%AE%A2%E9%98%85%E5%8F%B7.html)，本文使用了github上面sd4324530提供的[快速搭建微信公众平台服务器](https://github.com/sd4324530/fastweixin)。

## 前言
本文仅用于测试群发接口的可行性。

所以采用的main函数的方式

开始前请先登陆微信工作平台[测试号管理](https://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)，获取你的appID和appsecret



### 仅发送文本

```
import com.github.sd4324530.fastweixin.api.MessageAPI;
import com.github.sd4324530.fastweixin.api.config.ApiConfig;
import com.github.sd4324530.fastweixin.api.response.GetSendMessageResponse;
import com.github.sd4324530.fastweixin.message.TextMsg;

public class SendTest {
    private void sendAllMessage(ApiConfig config){
        TextMsg textMsg = new TextMsg();
        textMsg.setContent("My test message");//再次测试的时候需要更改消息内容后再测试
        MessageAPI messageAPI = new MessageAPI(config);
        GetSendMessageResponse messageResponse = messageAPI.sendMessageToUser(textMsg, true, 0);
        System.out.println(("Send Message Id is " + messageResponse.getMsgId()));
    }

    public static void main(String[] args) {
        SendTest sendTest = new SendTest();
        String appID = "wx****************";//这里填写你自己的appID
        String appsecret = "********************************";//这里填写你自己的appsecret
        ApiConfig config = new ApiConfig(appID, appsecret);
        sendTest.sendAllMessage(config);
    }
}
```




转载请标注原文链接
