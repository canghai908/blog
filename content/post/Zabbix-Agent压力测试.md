---
title: Zabbix Agent压力测试
date: 2018-01-26 16:14:35
tags: [zabbix, agent, 压力测试]
categories: [zabbix]
---

一直以来在 zabbix 群就经常见有人问到 Zabbix Server 到底能监控多少主机？如何做压力测试？Zabbx Agent 对主机性能有何影响?本文将为你介绍如何对 Zabbix Agent 进行压力测试。压力测试脚本为 py，下载地址https://dl.cactifans.com/zabbix/zabbix-agent-stress-test.py 下载之后最好是放到非压测机器，以免影响测试结果。本次我在 zabbix server 上测试 Agent 的压力情况。
使用./zabbix-agent-stress-test.py -h 可查看使用帮助

<!--more-->

![1](https://img.cactifans.com/wp-content/uploads/2018/01/8.jpg)
机器基本配置

| 指标        | 数值                |
| :---------- | :------------------ |
| 机器配置    | E5-2620 v3 8vcpu/8G |
| 操作系统    | CentOS 7.3.1611     |
| Agent 版本  | 3.4.4               |
| StartAgents | 3                   |

Agent 配置文件如下

```bash
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
Server=10.110.200.12
ServerActive=10.110.200.12
Hostname=mysqldb_16
```

Server 为我的 zabbix server 的 ip
测试命令如下

```bash
./zabbix-agent-stress-test.py -s 10.110.200.16 -k "system.run[sleep 1]" -t 4
```

10.110.200.16 为我已安装好的 Zabbix Agent 主机。
测试结果
![2](https://img.cactifans.com/wp-content/uploads/2018/01/10.jpg)
Agent 服务器负载
![3](https://img.cactifans.com/wp-content/uploads/2018/01/9.png)

通过以上方法对 agent 进行测试总体结果如下

| 测试 key            | 线程数 | StartAgents | qps     |
| :------------------ | :----- | :---------- | :------ |
| system.run[sleep 1] | 4      | 3           | 2677.98 |
| system.run[sleep 1] | 4      | 20          | 2727.12 |
| agent               | 4      | 3           | 2566.53 |
| agent               | 4      | 20          | 2612.39 |

经过以上测试，当 StartAgents 配置 3 时，每秒大约 2600 左右，agent 每个进程 cpu 占用率为 9%左右，内存占用很少。修改 StartAgents 线程时，qps 也大概在 2600 左右，没有明显提升。Agent 一般情况下不会对系统造成很大的影响，不过实际系统占用可能与你的自定义脚本有关系。可在https://github.com/monitoringartist/zabbix-agent-stress-test 查看更多细节
下期为大家介绍对 Zabbix Server 进行压力测试

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
