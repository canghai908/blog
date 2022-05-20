---
title: "Zabbix新版本数据库编码问题"
tags: [zabbix, 数据库]
categories: [zabbix]
date: 2020-04-24T16:21:19+08:00
---

### 现象

最近大家在安装新版本的 Zabbix 过程中，可能会经常看到这样的错误（图片来自 N 个群友）
![1](https://img.cactifans.com/wp-content/uploads/2020/04/WechatIMG284.png)

### 原因

在查看官网的 Release Notes 之后，发现新版本是增加了对数据库编码的检查，由于数据库编码不匹配，才导致以上问题，老版本无此问题

影响版本：ZABBIX 5.0.0alpha3,Zabbix 4.4.6 及 Zabbix 4.0.18 以后版本
![1](https://img.cactifans.com/wp-content/uploads/2020/04/F33F15B3-FB3C-4884-A6C1-49BE1D3255BF.png)
![1](https://img.cactifans.com/wp-content/uploads/2020/04/C1AA33FB-BBA8-44FA-AF03-44DECC6607C6.png)

具体查看 https://support.zabbix.com/browse/ZBXNEXT-5603

### 解决办法

如果为全新安装，建议删除 zabbix 数据库，使用以下命令重新创建数据库

```
create database zabbix character set utf8 collate utf8_bin;
```

之后导入对应 sql 文件即可解决。

### 总结

版本发布之后一定要查看官方的 Release Notes！！！
版本发布之后一定要查看官方的 Release Notes！！！
版本发布之后一定要查看官方的 Release Notes！！！
重要事情说三遍！！！
**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
