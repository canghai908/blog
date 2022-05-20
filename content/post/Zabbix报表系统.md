---
title: Zabbix报表系统
date: 2019-03-15 16:34:03
tags: [zabbix, 报表]
categories: [zabbix]
---

Zabbix 监控资源之后，常需要对资源的的监控数据进行导出，制作成为报表，如周报，日报等形式，目前 zabbix 还未自带报表功能。近期学习 go 语言，开发了一个简单的 Zabbix 报表工具。

## 在线试用

https://zbx.cactifans.com
直接点登录即可

## 系统截图

主机列表
![1](https://img.cactifans.com/wp-content/uploads/2019/03/929EB4EF-EB39-4134-B176-6D0E6AA7DA56.jpg)

数据导出
![2](https://img.cactifans.com/wp-content/uploads/2019/03/9FCF005B-912D-4FC6-BE68-FD8FDE5EED19.jpg)
报表展示
![3](https://img.cactifans.com/wp-content/uploads/2019/03/B8289CAF-591C-4A10-BF19-AA1530D3D2E0.jpg)
导出到 Excel
![4](https://img.cactifans.com/wp-content/uploads/2019/03/73605ABA-A3D2-490D-9BFA-DDADC5214749.jpg)
告警列表
![5](https://img.cactifans.com/wp-content/uploads/2019/03/260BA508-966C-4395-8311-2FCF38BD9B95.jpg)
告警统计
![6](https://img.cactifans.com/wp-content/uploads/2019/03/BE28FC69-4B39-4468-A991-985DC1B487F8.jpg)
告警导出到 Excel
![7](https://img.cactifans.com/wp-content/uploads/2019/03/3EF918DE-1BF2-40E5-9459-0C56882513F2.jpg)

## 基本情况

后端：go （beego web 框架 https://github.com/astaxie/beego ）
前端：vue （elment ui https://github.com/PanJiaChen/vue-admin-template）
绘图：Echart https://echarts.baidu.com/
Zabbix API： https://github.com/AlekSi/zabbix
数据库：MySQL 5.6

## 基本功能

1.可导入 Zabbix 监控的服务器(目前是 Linux&Windows)的 CPU 空闲率/内存 Free 空间/磁盘可用空间/网络流量，导出到 Excel 2.根据历史告警信息生成 Top10 告警主机，生成不同等级告警分类 3.根据主机名查询历史告警

## 开发思路

### 历史数据获取

1.基于 Zabbix API,通过调用 Zabbix API 的 hitory.get/item.get 获取指定指标的 itemid。
根据 hostid 和 item key 获取 items 具体信息

```
par := make(map[string]string)
par["key_"] = key
rep, err := API.Call("item.get", Params{"output": "extend", "sortfield": "name", "limit": "1", "hostids": hostid, "search": par})
```

2.根据 itemid 信息通过 trend.get 接口获具体指标的历史详情数据及趋势数据。
根据 itemid 获取 item 的趋势数据，limit 为输出条数

```
par := []string{"itemid", "clock", "num", "value_min", "value_avg", "value_max"}
par1 := []string{itemid}
rep, err := API.Call("trend.get", Params{"output": par,
		"itemids": par1, "limit": limit})
```

3.通过 Echart 绘图并导出到 Excel 报表

### 历史告警获取

历史告警通过 zabbix 的告警脚本功能发送到平台，解析之后存入数据库，进行分析

## 后续计划

目前还有平台功能较弱，后期打算新增以下功能： 1.支持导出任意监控指标的报表信息 2.支持导出历史详情信息,以监控指标采集周期为间隔 3.定制导出某 Linux 或 windows 机器的基本指标到一个 Excel，包括机器 CPU/内存/磁盘/网卡流量
4.Top 支持导出到 Excel 5.其他暂时没想到....

## 关于源码及使用

由于本人正在学习 go 语言及 vue，其代码质量太差，基本可以用下图来形容我的代码(真实合理)
![8](https://img.cactifans.com/wp-content/uploads/2019/03/WechatIMG21834.jpeg)
因此源码就不公开了。二进制软件后期会在公众号及 blog 公开下载地址，免费供大家使用(有可能是 License 授权码方式)。

欢迎在下方留言，提出合理建议及需求。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
