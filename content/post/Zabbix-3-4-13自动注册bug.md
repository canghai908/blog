---
title: Zabbix 3.4.13 自动注册 Bug
date: 2018-09-18 10:41:51
tags: [bug, zabbix]
categories: [zabbix]
---

## 问题描述

最近在使用 Zabbix3.4.13 版本中，被监控机器配置好 Agent 之后，并在 Zabbix server 中建立如下自动注册规则之后,发现机器没有注册,
![enter image description here](https://img.cactifans.com/wp-content/uploads/2018/09/5DDA1A2D-1873-457C-8990-56F80889D6E2.jpg)
在 Zabbix Server 日志中会出现如下错误
![enter image description here](https://img.cactifans.com/wp-content/uploads/2018/09/D1618EA5-6385-4D96-BDC6-7C42353888F0.jpg)

### 解决方案

最后在 Zabbix 论坛找到确认为 bug：
https://www.zabbix.com/forum/zabbix-troubleshooting-and-problems/365212-zabbix-autoregistration-cause-an-error-when-linked-to-template 1.官方在 3.4.13rc1 之后修复，说明
https://support.zabbix.com/browse/ZBX-14614 2.取消关联模版，即完成添加主机

### Tips

自动注册动作里，不需要添加主机，直接添加到组，关联模版即可。官方也有对应的说明。
地址：https://www.zabbix.com/documentation/3.4/manual/discovery/auto_registration
说明：
![enter image description here](https://img.cactifans.com/wp-content/uploads/2018/09/5E9DF830-441A-4464-AB3B-C78C9DAF32B4.jpg)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
