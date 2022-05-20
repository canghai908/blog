---
title: ZABBIX4.0.0alpha6新特性体验
date: 2018-05-24 14:30:56
tags: [Json, zabbix, 4.0, Http agent]
categories: [zabbix]
---

ZABBIX 于 4 月 27 日推出了 4.0.0alpha6，在 Release Notes 里的 New Features 里看到里不少 Features

> ZBXNEXT-4417
> added real time export of events, history and trends in newline delimited JSON format
> ZBXNEXT-4458
> improved logging of Java gateway, added username/password validation for JMX items
> ZBXNEXT-4411
> added compression of server-proxy data exchange
> ZBXNEXT-4488
> added ability to push data via trapper to HTTP agent item type
> ZBXNEXT-4358
> added HTTP agent item type for data gathering via HTTP

基于以上众多的特性，安装一套来体验

### Json 数据输出

可输出 zabbix 事件/历史数据/趋势数据到 json 文件。Zabbix Server 配置里增加里一个配置数据导出目录和导出文件大小

```
### Option: ExportDir
#       Directory for real time export of events, history and trends in newline delimited JSON format.
#       If set, enables real time export.
#
# Mandatory: no
# Default:
ExportDir=/opt/exp

### Option: ExportFileSize
#       Maximum size per export file in bytes.
#       Only used for rotation if ExportDir is set.
#
# Mandatory: no
# Range: 1M-1G
# Default:
ExportFileSize=50M
```

ExportDir 为导出数据保存的目录，这里填绝对路径，并且需要有 zabbix 权限
ExportFileSize 为导出文件大小设置
这里配置的输出目录为/opt/exp,配置所有者为 zabbix

```bash
chown -R zabbix:zabbix /opt/exp
```

配置之后，重启 zabbix server 即可在目录看到文件
![1](https://img.cactifans.com/wp-content/uploads/2018/05/D99353CD-480F-4EED-827C-B020034EEF6A.jpg)
文件内容
![2](https://img.cactifans.com/wp-content/uploads/2018/05/DB9BCFDD-316D-488A-9DD7-18F4A0AEC216.jpg)
文件内容为每行一条 json 数据。

##### 扩展

可以通过程序读取，输出到 Kafka，MQ，Es 等，并写入到其他的数据库，也可以做实时的数据处理。与本人去年 zabbix 大会上对历史数据的处理有异曲同工之处。哈哈！
![3](https://img.cactifans.com/wp-content/uploads/2018/05/718C1781-374F-4912-94F7-F53F033EDCB7.jpg)

### Http Agent

在 Agent 的类型里增加里一种 Http Agent，尚不清楚是什么时候添加过来的，和之前的 web 检测有类似之处，这里主要是通过 Agent 去请求 URL，去采集数据直接写入数据库。
点击主机，创建一个 item，类型选择 http Agent，可以看到选项比较多，
![4](https://img.cactifans.com/wp-content/uploads/2018/05/C8356BEF-5C53-4F1E-96ED-5F4A9ADAF985.jpg)
![5](https://img.cactifans.com/wp-content/uploads/2018/05/25E7C7CE-996B-4844-A4B6-0B731B4C2317.jpg)

主要有以下特性 1.支持 get/post/put/head 等请求方法，支持 raw/json/xml 为参数 2.支持代理访问 3.支持 Basic/NTLM/SSL 等认证
这里配置一个获取网站内容的 item，配置之后可以获取到内容
![6](https://img.cactifans.com/wp-content/uploads/2018/05/C619B745-4196-42BB-9832-13D92200E10F.jpg)

##### 扩展

通过此功能，可以对一些接口的数据进行采集，同时可以监控接口状态，比 web 检测功能丰富

## 结语

以上众多新特性使我们很期待 4.0 的正式发布！由于为 alpha 版本，切勿用于生产环境。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
