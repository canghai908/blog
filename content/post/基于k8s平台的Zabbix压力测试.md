---
title: 基于k8s平台的Zabbix压力测试
date: 2019-01-22 15:03:23
tags: [k8s, zabbix, 压力测试]
categories: [zabbix]
---

本文以 2019 年 1 月 16 日 Webinars 课程内容编写
内容目录
![1](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8703.png)
视频下载地址:

https://we.tl/t-A2ENyLjvCW

### k8s 介绍

![2](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8704.png)
kubernetes，简称 K8s，是用 8 代替 8 个字符“ubernete”而成的缩写。Kubernetes 是 Google 开源的一个容器编排引擎，它支持自动化部署、大规模可伸缩、应用容器化管理。
k8s 架构
![3](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8705.png)
本次环境及配置
![4](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8706.png)
机器为虚拟机，磁盘为普通的 SAS 硬盘做 RAID1，性能不是太好。
zabbix docker 官方仓库，目前包括以下镜像
![5](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8708.png)
根据不同需要，选择使用，具体使用方法请查看我之前一篇的文章：[Zabbix 在 Docker 中的应用和监控](https://blog.cactifans.com/2018/12/28/Zabbix%E5%9C%A8Docker%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8%E5%92%8C%E7%9B%91%E6%8E%A7/)

### Zabbix 部署

部署在 k8s 平台的 zabbix 架构
![6](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8712.png)
zabbix server 和 zabbix web 以 k8s service 形式提供。同时部署二个 zabbix web pod 进行负载均衡，zabbix server 只运行一个 Pod。
所使用的 yaml 文件
![8](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8711.png)

> zabbix-agent-stress 为添加了压测模块的 agent
> zabbix-server-mysql 在 zabbix4.0.3 版本的 zabbix server
> zabbix-secret 为 zabbix 连接数据的账号及密码，采用 k8s secret 保存，默认账号:zabbix 默认密码:zabbixpwd123
> zabbix-svc 文件为 zabbix server 及 zabbix web 创建 k8s service，提供给外部访问
> zabbix-web-nginx-mysql 为 zabbix 前端部署文件

使用的镜像：
hub.c.163.com/canghai809/zabbix-server-mysql:v4.0.3
hub.c.163.com/canghai809/zabbix-web-nginx-mysql:v4.0.3
hub.c.163.com/canghai809/zabbix-agent-stress:v4.0.3
使用的 yaml 文件:https://dl.cactifans.com/zabbix/zabbix-kubernetes-yaml.tar.gz

#### 数据库导入

下载对应版本的 zabbix 源码，本文以 4.0.3 为例子，并创建 zabbix 数据库文件 create.sql

```bash
cat database/mysql/schema.sql > create.sql
cat database/mysql/images.sql >> create.sql
cat database/mysql/data.sql >> create.sql
```

导入 zabbix 数据库文件 create.sql 到数据库服务器，并建立用户并授权

```sql
create database zabbix;
grant all on zabbix.* to zabbix@'%' identfied by 'zabbixpwd123';
use zabbix；
source /opt/create.sql；
```

#### 使用 yaml 部署

下载部署的 yaml 文件之后在 k8s master 上执行以下命令进行部署

```
wget https://dl.cactifans.com/zabbix/zabbix-kubernetes-yaml.tar.gz
tar zxvf zabbix-kubernetes-yaml.tar.gz
kubectl apply -f zabbix/zabbix-svc.yaml
kubectl apply -f zabbix/
```

即可一键部署 zabbix，即可部署完成。
部署成功后的状态如下
![9](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_224509.png)
查看 Service
![10](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_224547.png)
可使用任何一个节点 ip:30080 访问 zabbix web，默认账号:Admin 密码:zabbix
![11](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_224701.png)

### Zabbix 压力测试

zabbix 日常使用中的疑问
![7](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8715.png)
在 zabbix 的使用过程中，以下问题是大家经常关心的：
1.zabbix server 每秒可以处理多少指标？
2.zabbix agent 的性能到底如何？ 3.如何规划 zabbix 数据库的磁盘？
对于这方面的疑问目前较多，网上相关的教程及资料较少，因此依赖与 k8s 的自动化部署及高伸缩特性，通过不断在增加 zabbix agent，结合压力测试模块对 zabbix server 及 zabbix agent 进行压力测试。
基本架构:
![8](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8717.png)
通过 k8s 的 deploy 模式部署 zabbix agent，通过 zabbix 的自动注册功能，添加到指定的组并关联指定的压力模版
zabibx 压力测试模块及模版:
![7](https://img.cactifans.com/wp-content/uploads/2019/01/%E5%B9%BB%E7%81%AF%E7%89%8716.png)
github 地址为:https://github.com/monitoringartist/zabbix-server-stress-test
提供了压力测试的 module，编译之后，利用 zabbix module 功能，加载到 zabbix agent，重启 agent 即可添加。template 目录下提供了压力测试模版：分为主动和被动二种模式，模版每有 1k 个 item，每秒采集一次。还有 2 个每个包含 5k 个 item 的(item 过多，不建议使用)模版，导入到 zabbix server。自动发现添加 zabbix 之后关联对应的压力测试模版即可。
在 zabbix 上导入模版，分别导入二个模版

> Template App Zabbix Server Stress 1k active A.xml 主动模式压力测试模版
> Template App Zabbix Server Stress 1k passive A.xml 被动模式压力测试模版

#### Zabbix 相关参数

以下为本次测试所配置的 Zabbix Server 参数，其他参数为 zabbix docker 默认参数

```
- name: ZBX_CACHESIZE
  value: "1024M"
- name: ZBX_TRENDCACHESIZE
  value: "1024M"
- name: ZBX_HISTORYCACHESIZE
  value: "2048M"
- name: ZBX_HISTORYINDEXCACHESIZE
  value: "1024M"
- name: ZBX_STARTTRAPPERS
  value: "50"
- name: ZBX_STARTPREPROCESSORS
  value: "300"
- name: ZBX_STARTDBSYNCERS
  value: "100"
```

zabbix db 为使用 rpm 包安装，mysql server 版本为 mysql-community-server-5.7.24-1.el7.x86_64 配置参数为

```
lower_case_table_names=1
character_set_server=utf8

wait_timeout=31536000
interactive_timeout=31536000
max_connections=1000
validate_password_policy=LOW
```

#### 主动模式测试

建立自动注册动作,并关联主动模式模版，如图：
![11](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_233158.png)
![12](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_233650.png)
关联主动模式测试模版
![12](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_233757.png)
添加成功之后如下图
![12](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_233857.png)
添加之后过一会即可看到主机已自动添加到 zabbix server
![13](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_234500.png)
Zabbix Server 及 DB 性能
Dashboard
![2](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-21_234516.png)
Zabbix Performance Overview
![13](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_003537.png)
Db iotop
![23](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_003642.png)
DB TOP
![22](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_003712.png)

#### 被动模式测试

测试主动模式之后，先停止自动注册动作，删除所有 agent 节点，并修改自动注册关联被动模式模版，并启用，如图：
![23](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_004140.png)
过一会即可看到 agent 已自动添加到 zabbix server 中并关联了被动压力测试模版
![33](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_004651.png)
Zabbix Server 及 DB 性能
Dashboard
![32](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_010736.png)
Zabbix Performance Overview
![23](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_004839.png)
Db iotop
![32](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_004850.png)
Db top
![34](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_004911.png)

### 总结

综合对比，如下图：
![32](https://img.cactifans.com/wp-content/uploads/2019/01/2019-01-22_012501.png)
折线处为从主动模式更换为被动模式

经过以上对 zabbix server 及 agent 的简单压力测试，可得出以下结论：

#### 1.Zabbix Server 压力

主动模式 Zabbix Server 压力明显较小，被动模式对 zabbix 压力较大，nvps 到达 5k 后，50 个 poller processes 已经 100%，需要增加 poller 采集数量，后期测试 poller 增加到 300 后，poller processes 占用为 54%，网络流量 15Mbps，DB 及 Server 负载变高，Zabbix Server 运行稳定。
**结论：建议使用主动模式。**

#### 2.DB 压力

zabbix 性能瓶颈主要在数据库，主动模式下数据库写入流量一般，机器负载较低，被动模式下，数据库负载较高，对 mysql 的操作较为频繁。5k nvps 情况下数据库写入最大 13MB/s，写入较为正常，出现少量 iowait 情况(机器为 SATA 磁盘性能较差)，如增大，有可能出现告数据写入延迟，告警异常情况。
**结论：实时观察 DB 压力情况，进行优化，实行分区表，关闭 zabbix 自带的 housekeep 工具**

#### 3.压力估算

机器估算：每秒为 5k nvps。每分钟可处理 5kX60=30w 指标，如每个机器有 50 个指标，每分钟采集一次，**此压力测试约为 30w/50=6k 台机器的压力情况**。
空间估算：分钟约 20MB 左右磁盘占用，如保留一月数据，则需要：20x60x24x30=**0.8T 空间**。
以上估算由于收到网络/硬件/策略/指标类型等影响，只做参考使用。
**结论：历史数据过多会造成页面打开缓慢，擦操作缓慢等情况，建议根据实际情况保留历史数据，或抽取历史数据到其他库，以此来减轻主库压力，建议采用分区表，关闭 zabbix server 自带 housekeeper 清理。**

#### 4.Zabbix Agent 压力

Zabbix Agent 在 1k item 压力情况,主机 8vcpu/16G 性能数据如下：

| 模式     | CPU 占用(%) | 内存(MB) |
| -------- | ----------- | -------- |
| 普通模式 | 0.02-0.03   | 2.2-3    |
| 主动模式 | 1.5-2.1     | 9.5-10   |
| 被动模式 | 3-5         | 25-30    |

在主动或被动模式下，Zabbix Agent 均未出现负载过高情况影响系统的情形，也未出现假死等情况，所以 Zabbix Agent 的稳定性大家不用怀疑。众多情况问题都源于用户的自定义脚本，用户自定义脚本性能问题，影响系统，因而使 Zabbix 被动"背锅"
**结论:Zabbix agent 较为稳定，建议优化自定义脚本性能。**

### 结语

以上为简单的测试，还未测试 proxy 组件及数据库主动模式下的情况，后续将会推出。
