---
title: zabbix3.4.7版本饼图显示问题
date: 2018-04-08 09:35:11
tags: [zabbix, 饼图, 空白]
categories: [zabbix]
---

## 问题描述

最近使用 zabbix3.4.7 版本，发现监控 Linux 的主机关联系统自带的 Template OS Linux 模版之后，磁盘空间饼图显示有问题，出现空白，如图所示
![](https://img.cactifans.com/wp-content/uploads/2018/04/1.png)

## 解决方案

虽然本人不怎么看图，由于询问人数过多，就看了一下，查看之后，确定为自带的 Lemplate OS Linux 模版问题，修复方式如下
点击**Templates**（模版）找到**Template OS Linux**模版，点击**Discovery rules**（自动发现规则）在**Mounted filesystem discovery**（挂载文件系统自动发现）里点击**Graph prototypes**（图形原型）
![](https://img.cactifans.com/wp-content/uploads/2018/04/2.jpg)
点击**Disk space usage**
![](https://img.cactifans.com/wp-content/uploads/2018/04/3.jpg)
如图所示，将**Template OS Linux: Total disk space on**的**Type**（类型）修改为**Simple**（简单）之后点击**Update**更新模版即可
![](https://img.cactifans.com/wp-content/uploads/2018/04/4.jpg)
修改模版之后，过 1 小时之后（模版里自动发现默认为 1 小时），图形会显示正常。
![](https://img.cactifans.com/wp-content/uploads/2018/04/5.jpg)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
