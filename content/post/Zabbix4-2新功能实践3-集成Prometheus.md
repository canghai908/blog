---
title: Zabbix4.2新功能实践3-集成Prometheus
date: 2019-04-26 11:29:03
tags: [zabbix, prometheus]
categories: [zabbix]
---

zabbix 能够以多种不同的方式（推/拉）从各种数据源收集数据，包括 JMX，SNMP，WMI，HTTP / HTTPS，RestAPI，XML Soap，SSH，Telnet，代理，脚本和其他数据源，4.2 版本支持了 Prometheus 数据源。使用单个 HTTP agnet 调用获取所有数据，通过依赖指标高效的收集大量的 Prometheus 指标，然后仅将其用于相关指标监控，还可以将 Prometheus 数据转换为 JSON 格式，直接用于低级别发现。

### Prometheus Exporter

Prometheus 提供了基本的采集客户端称为: Exporter,下载对应的 Exporter 运行，采集指标通过 http 暴露。以采集主机信息的 node_exporter 为例

#### 安装

以 Linux node_exporter 为例
下载并运行

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz
tar zxvf node_exporter-0.17.0.linux-amd64.tar.gz
cd node_exporter-0.17.0.linux-amd64
./node_exporter
```

![1](https://img.cactifans.com/wp-content/uploads/2019/04/724A2C74-BA09-4C51-A880-321A426431EE-1024x618.jpg)
表示启动成功，访问http://Ip:9100/metrics 可以看到所有采集的 Metrics，安装成功。
![11](https://img.cactifans.com/wp-content/uploads/2019/04/19E4316E-7F3D-44F3-A023-91BD880A84F4.jpg)

#### 数据结构

通过 http 可以看到所有的 metric，metric 有固定的数据结构
![2](https://img.cactifans.com/wp-content/uploads/2019/04/AB9652F8-5FDF-4933-8807-E31C76876D55.jpg)
主要分为以下几个部分

##### 说明

以#号开头 HELP metric 的说明，前面为 metric 名称,空格后为说明

##### 类型

以#号开头 TYPE metric 的类型，前面为 metric 名称,空格后为类型
Metrics 类型有四种

| 指标类型  | 描述                                             | 说明                                                               |
| :-------- | :----------------------------------------------- | :----------------------------------------------------------------- |
| Counter   | 只增不减的计数器，其值只能增加或在重启时重置为零 | 例如，您可以使用计数器来表示服务的总请求数，已完成的任务或错误总数 |
| Gauge     | 用来存放一个可以任意变大变小的数值               | 例如温度或当前内存使用情况，或者运行的 goroutine 数量              |
| Histogram | 一段时间范围内对数据进行采样                     | 通常用它计算分位数的直方图                                         |
| Summary   | 客户端定义的数据分布统计图                       | 统计事件发生的次数或者大小，以及其分布情况                         |

其中 Counter 和 Gauge 最为常用

##### 数据

数据结构
![3](https://img.cactifans.com/wp-content/uploads/2019/04/5266AFB9-63CB-4E92-A2C1-0C4F677FEBA2.jpg)

### 集成 Prometheus

Prometheus 的 Exporter 为 http 方式，因此需要使用 Zabbix 的 http agent，配合使用 Zabbix 的 Dependent items 做到一次采集所有指标。由于 Prometheus metric 较为通用，建议配置独立的模版。
建立一个名为 Templage Prometheus 的模版，添加一个 Master Item
![4](https://img.cactifans.com/wp-content/uploads/2019/04/843B2CC6-5EA4-49FA-B267-CAF945753E37.jpg)
关键配置
![5](https://img.cactifans.com/wp-content/uploads/2019/04/7372D0AD-B2C5-4331-87CA-FD4A74FFA6F1.png)
配置 node_exporter 的地址为宏变量
![6](https://img.cactifans.com/wp-content/uploads/2019/04/A6E9FD9C-9E5B-4D3A-AF22-2728CDE80B13-1024x373.jpg)

#### 一般采集

配置好之后，配置一个监控操作系统 Load5 的 Item
新建 Item
![7](https://img.cactifans.com/wp-content/uploads/2019/04/73DEF247-3CEE-4C22-87C6-6E2EEF920D75.jpg)
配置之后要配置数据预处理策略
![8](https://img.cactifans.com/wp-content/uploads/2019/04/B007D56B-11B9-4C9F-AA0C-40EC95071217-1024x384.jpg)
配置之后关联到主机，之后查看数据，已经采集
![9](https://img.cactifans.com/wp-content/uploads/2019/04/2E3B7526-278F-4D05-99FF-D8870DDD67DC-1024x275.jpg)

#### LLD（低级别发现）

使用 zabbix agent，用 LLD 可实现自动发现磁盘空间，网卡等不定项的指标，利用 LLD 也可以 Prometheus 指标的发现。本次以配置自动发现网卡流量为例。
在模版里配置发现规则
![10](https://img.cactifans.com/wp-content/uploads/2019/04/DE357DB6-1822-4997-A837-627CD8EF6AD8-1024x535.jpg)
配置数据预处理
![11](https://img.cactifans.com/wp-content/uploads/2019/04/CFB1A21A-44B3-4E6D-B9D9-25BA412B5DFC-1024x270.jpg)
这里使用如下 metrics，

```
# HELP node_network_device_id device_id value of /sys/class/net/<iface>.
# TYPE node_network_device_id gauge
node_network_device_id{interface="enp0s3"} 0
node_network_device_id{interface="enp0s8"} 0
node_network_device_id{interface="lo"} 0
```

使用通配符获取所有网卡。通配符及语法使用方法查看https://www.zabbix.com/documentation/4.2/manual/config/items/itemtypes/prometheus
parameters 配置为

```bash
node_network_device_id{interface=~".*"}
```

配置之后，复制一个 metric（完整的 metric，包括 value），点击 test all steps，粘贴 metric，点击 test
![12](https://img.cactifans.com/wp-content/uploads/2019/04/C953EC55-E275-41E4-ABC9-AD906076D46E-1024x307.jpg)
如提示错误，表示规则配置有问题，需要修改。如没有错误，下方会出现处理后的 json，复制 json 文本，格式化
![13](https://img.cactifans.com/wp-content/uploads/2019/04/9D082B32-B383-405B-84FF-434A9649770B.jpg)
根据 json 格式配置如下宏
![14](https://img.cactifans.com/wp-content/uploads/2019/04/068D1DEA-136C-4F03-A8B4-8679E91DE54E.jpg)
提取 lables 里的 interface 即网卡名称作为宏
配置如下过滤规则，过滤 lo 网卡
![15](https://img.cactifans.com/wp-content/uploads/2019/04/AD878175-DED0-4F07-9737-948CAEA9B4DE-1024x320.jpg)
网卡分为发送和接收 2 个方向，因此需要创建 2 个基本的 Item 发现原型
配置网卡接收 item
![16](https://img.cactifans.com/wp-content/uploads/2019/04/0A6E7AF0-4A0E-4349-96F1-187881E056AB-1024x580.jpg)
Prometheus 的网卡接收 Metrics 如下

```bash
# HELP node_network_receive_bytes_total Network device statistic receive_bytes.
# TYPE node_network_receive_bytes_total counter
node_network_receive_bytes_total{device="enp0s3"} 5.703155e+06
node_network_receive_bytes_total{device="enp0s8"} 6.303864e+06
node_network_receive_bytes_total{device="lo"} 3.00503e+07
```

Prometheus pattern 参数配置为

```bash
node_network_receive_bytes_total{device="{#INTERFACE}"}
```

由于网卡流量为 bytes，Metric 类型 counter，配置数据预处理，如图
![17](https://img.cactifans.com/wp-content/uploads/2019/04/58F8AFC3-429A-4158-BAF5-D9F564FAC9D3.jpg)
同样配置配置，网卡发送
![18](https://img.cactifans.com/wp-content/uploads/2019/04/66F811A8-DD65-4EEB-A09B-9387013B18A8-1024x614.jpg)
metrics

```
# HELP node_network_transmit_bytes_total Network device statistic transmit_bytes.
# TYPE node_network_transmit_bytes_total counter
node_network_transmit_bytes_total{device="enp0s3"} 1.4358114e+07
node_network_transmit_bytes_total{device="enp0s8"} 9510
node_network_transmit_bytes_total{device="lo"} 3.00503e+07
```

Prometheus pattern 参数配置为

```bash
node_network_transmit_bytes_total{device="{#INTERFACE}"}
```

配置同样的预处理规则
![19](https://img.cactifans.com/wp-content/uploads/2019/04/C6459A79-2A19-422D-AA4B-5C52E16A77D6-1024x437.jpg)
注意配置 2 个 Item 的 key 不能重复
最后创建一个图形原型
![20](https://img.cactifans.com/wp-content/uploads/2019/04/50E8BE32-1E08-42A3-BE66-7A124F32253D-1024x663.jpg)
配置完成之后，可以看到数据已采集
最新数据
![21](https://img.cactifans.com/wp-content/uploads/2019/04/377F6423-01C4-41BD-B700-A12C9EC0F39A-1024x331.jpg)
图形
![22](https://img.cactifans.com/wp-content/uploads/2019/04/45C5A85A-8E0E-4061-9771-29BAAAE2C72A-1024x417.jpg)

### 总结

Zabbix4.2 提供了很多新的功能及特性，对 Prometheus 的支持可以整合现有的 Prometheus 监控资源，利用 Throttling 等功能可以做到高效的资源监控。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
