---
title: Zabbix 4.0新功能介绍1-防止单位转换
date: 2018-10-12 08:58:06
tags: [zabbix4.0, 新功能, 单位转换]
categories: [zabbix]
---

zabbix4.0 LTS 版本已经在国庆期间发布，带来众多新特性及功能，最近会陆续推出 4.0 的一些功能介绍文章，今天为第一篇——防止单位转换

### 原有方式

在 4.0 之前，如某个 ITEM 的数据大于 1000，在 Graph 里就会展示成 1k，zabbix 会自动对数据进行单位转换，诸如此类。此方式可避免过大的数据展示在页面同时方便查看，但同时也带来一个问题：如果需要具体查看某个数据的小的变化，就不能了，因此有很多同学就提出能不能大于 1000 不自动转换单位？在 4.0 之前版本是没有解决方式的。

### 现有方式

在 4.0 里，解决了大家的这个需求，可以对 ITEM 的单位进行配置，配置为不自动转换单位，既可显示具体的数据。具体配置方式为在 ITEM 的配置里在 Units 里添加！号即可。下面以内存为例子：
配置：
![1](https://img.cactifans.com/wp-content/uploads/2018/10/8170E580-047F-4B6D-A5D9-012C6E3732AB.jpg)
最新数据
![2](https://img.cactifans.com/wp-content/uploads/2018/10/73754AC4-272A-4F04-B8E6-D0372293FA7E.jpg)
图形
![3](https://img.cactifans.com/wp-content/uploads/2018/10/EB0B3A75-C94B-4370-8B69-0BFE10884A70.jpg)

可以看到已经显示具体的数值了。也可以在原有单位前面加！号也不会自动转换单位，举例如下：

> 1024 !B -> 1024 B
> 1024 B -> 1 KB
> 61 !s -> 61 s
> 61 s -> 1m 1s
> 0 !uptime -> 0 uptime
> 0 uptime -> 00:00:00
> 0 !! -> 0 !
> 0 ! -> 0

### 结语

官方文档也有具体的例子及说明
![4](https://img.cactifans.com/wp-content/uploads/2018/10/8CF264BD-CD17-4917-9232-0324BF26734F.jpg)
具体使用方式可查看官方文档说明：[https://www.zabbix.com/documentation/4.0/manual/introduction/whatsnew400](https://www.zabbix.com/documentation/4.0/manual/introduction/whatsnew400)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
