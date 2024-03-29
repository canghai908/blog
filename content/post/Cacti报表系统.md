---
title: Cacti报表系统
date: 2018-04-19 20:37:01
tags: [cacti, 报表]
categories: [cacti]
---

本系统是结合 Cacti 监控，通过读取 Cacti 监控数据，导出指定设备的详情数据。

## 设计初衷

Cacti 是一套基于 PHP,MySQL,SNMP 及 RRDTool 开发的开源网络流量监测图形分析工具。主要给予一下原因考虑，开发设计此系统

1. 如果需要导出某一设备一段时间内的详情数据，Cacti 默认只能导出 2 天左右的详情数据（默认为 5 分钟一采集）。如果需要修改，可参考此文章http://www.cactifans.org/cacti/1564.html 修改，修改之后需要删除所有 RRD 文件（Cacti 数据存储，所有**历史监控数据将会丢失**）
2. 如果需要批量导出某些设备的监控数据，在页面操作就会比较麻烦

## 基本操作

#### 1. 添加报表

默认不会读取监控数据。如果采集某一设备的数据并做报表，首先要将该设备需要报表的图形添加到报表
点击【报表设置】--选择设备会出现该设备所有的监控图形，选择需要出报表的图形，点击【添加到报表】，添加成功之后，数据就会开始采集，默认为 1 小时读取一次

#### 2. 报表导出

添加到报表之后，后台程序会自动读取该监控图形的数据，并写入数据库存储。点击【批量报表】选择设备之后，会展示该设备下面添加到报表的所有图形以及已经采集的历史数据时间，以小时为单位。如需要导出数据可选择时间，选择图形之后，点击【导出报表】之后点击【下载报表】就会将所有的图形报表打包成 zip 下载

#### 3. 日常查询

如需要查询某一图形的数据及信息，可点击【报表查询】，选择对应图形及时间段之后，点击【查询】导出独立的报表

## Tips

1. 不添加到报表，系统不会采集数据，自然就无法导出报表数据
2. 导出文件为 xlsx 文件，需要用 Excel 打开
3. 如经常导出报表，会在程序的 download 下生成导出文件，可自行清理，对系统无影响
4. 默认账号：admin，默认密码：admin，可在后台修改
