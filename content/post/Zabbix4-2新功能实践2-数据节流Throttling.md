---
title: Zabbix4.2新功能实践2-数据节流Throttling
date: 2019-04-19 14:09:20
tags: [4.2, Throttling]
categories: [zabbix]
---

Zabbix4.2 增加了一个 Item 预处理功能:Throttling（节流）功能。通过此功能可以实现以下几个效果： 1.减少 Item 重复数据的存储。 2.对高频率采集数据进行压缩存储。
总结起来就是可以减少 Item 采集的重复数据存储，具体使用方法及用途通过以下几个实验说明

### 配置 Throttling

配置 Item 的 Throttling 功能，可在 item 的预处理配置
![1](https://img.cactifans.com/wp-content/uploads/2019/04/245A116D-66DD-484A-B779-D72B8F52AFF8.jpg)

分别有二个选项

> Discard unchanged 丢弃不变化的数据
> Discard unchanged with heartbeat 带心跳检查丢弃不变化的数据

Discard unchanged 为直接丢弃重复的数据，如 item 采集的前一个数据和目前数据重复，则只保存前一个数据，直接丢弃后续采集的数据
Discard unchanged with heartbeat 为配置一个心跳时间，此时间内至少会存储一个不变的采集数据

### 配置 Discard unchanged

以我之前写的 agent 为例,目前有一个 Item 为采集服务器 cpu 型号的 item 如下
![2](https://img.cactifans.com/wp-content/uploads/2019/04/BC58C421-BC69-4368-AF37-1B69626A39D5.jpg)
配置 Throttling 如下
![3](https://img.cactifans.com/wp-content/uploads/2019/04/D2E67821-A1F6-4B68-A406-536A4705D2E0.jpg)
现在改变 item 的采集数据
![4](https://img.cactifans.com/wp-content/uploads/2019/04/C857859E-9B93-4A0C-925D-5731A86AE748.png)
数据变化后，Throttling 功能没有生效！！！Item 依然保存数据.如采集数据不发生变化时，只存储了第一个数据，后续数据已不再存储。查看 Agent 日志如下
![5](https://img.cactifans.com/wp-content/uploads/2019/04/6C24B3D0-7EB7-461C-B9AA-9C330A1DCAF8-1024x509.jpg)
显示 Zabbix Server 还在采集数据,说明数据已采集到 zabbix server 只是丢弃了，没有进行存储。

### 配置 Discard unchanged with heartbeat

配置 Discard unchanged with heartbeat
![6](https://img.cactifans.com/wp-content/uploads/2019/04/46F0A227-B2B3-47A6-87CC-4705637651D4.jpg)
如下配置之后，Item 的采集周期为 1 分钟，心跳配置为 5 分钟，如数据不发生变化，每 5 分钟会有一个数据被存储,由于数据重复，可视为对重复数据进行了压缩存储。
![7](https://img.cactifans.com/wp-content/uploads/2019/04/BC4E788E-4467-449E-BCCB-A62D979BBBE4.jpg)
如数据发生变化时，可以看到配置的策略已不生效，数据按照 Item 采集周期被采集和存储。
![8](https://img.cactifans.com/wp-content/uploads/2019/04/E21DDE73-78BB-4478-A803-0A1EE744557D.jpg)
如配置 Discard unchanged 的时间为 30 秒
![9](https://img.cactifans.com/wp-content/uploads/2019/04/BA99EB80-E535-4995-BD19-291538ED2FAF.jpg)
即小于 Item 采集周期，数据还是以 Item 的采集周期为准，进行采集和存储。
![10](https://img.cactifans.com/wp-content/uploads/2019/04/4EE0303A-C710-4DF4-814E-5E1525C41B95.jpg)

### 结论

1.配置 Discard unchanged 之后，如采集数据发生变化，Throttling 配置不生效，正常采集存储数据。数据不变化时,采集正常执行，但只存储一个数据，但不影响告警等功能。 2.配置 Discard unchanged with heartbeat 之后，在心跳周期内至少存储一个数据，如数据发生变化，则配置的心跳时间不生效，以指标采集周期为准,采集存储数据。

### 使用场景

1.高频率采集情况下，如秒级别采集，配置合适的心跳，即能保证不错过变化的采集数据和告警，又能降低数据存储压力。 2.对一般采集数据不发生变化的采集指标，配置之后可起到数据“压缩”功能，降低数据存储压力。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
