---
title: 使用Golang实现一个简单的Zabbix Agent
date: 2019-03-29 12:36:51
tags: [golang, zabbix]
categories: [zabbix, golang]
---

Zabbix 提供了开放的协议，因此可以根据协议，实现自定义的 Zabbix Agent。Zabbix 自带的 Agent 已经很很稳定，而且基本可以做到全平台的适配,建议使用官方 Agent，这里只是学习，了解 Zabbix 协议，探索使用。

## Zabbix 协议

在 Zabbix 官网文档中，可以找到 Zabbix 的协议详细说明，以及数据交互的详细过程。

### Header Protocol

Zabbix 通信的头部协议结构: https://www.zabbix.com/documentation/4.0/manual/appendix/protocols/header_datalen 每次通信都需要的。这个协议在 zabbix 4.0 版本中进行了更新，因此导致部分第三方不标准的采集客户端无法使用，这里就包括了使用普遍:Orabbix，部分人升级到 Zabbix 到 4.0 之后不能使用此组件，就是这个原因导致，具体修复方法请查看这个 issue https://github.com/smartmarmot/DBforBIX/issues/62

### Passive and active agent checks

主动与被动检查协议：https://www.zabbix.com/documentation/4.0/manual/appendix/items/activepassive 此协议介绍了如何实现 Agent 主动与被动检查，以及具体的数据交互过程

### Zabbix sender protocol

Zabbix Trapper 采集协议：https://www.zabbix.com/documentation/4.0/manual/appendix/items/trapper 使用此协议可以实现 zabibx_sender 命令的功能，之前我已经使用此协议实现了一个 mysql 监控工具，工具介绍：https://blog.cactifans.com/2018/08/08/Zabbix%E7%9B%91%E6%8E%A7MySQL%E5%B7%A5%E5%85%B7/ 源码：https://github.com/canghai908/zabbix-mymon

## Agent-go 实现

了解 Zabbix 协议之后，就可以根据协议使用不同语言实现 Zabbix Agent 的各种功能。这里使用 https://github.com/fujiwara/go-zabbix-get 实现一个支持被动检查的 Agent，源码及打包好的 Agent 的 centos 7_x64rpm 包
**源码地址**：https://github.com/canghai908/agent-go.git
**RPM 包**：https://dl.cactifans.com/tools/agent-go-0.0.1-1.el7.x86_64.rpm  
**模版**：https://dl.cactifans.com/tools/template_linux_agent_go.tar.gz

```
3c39e829e1611543170b3f26fad128a123270234  agent-go-0.0.1-1.el7.x86_64.rpm
```

目前一共实现里 6 个监控指标的被动采集，数据采集使用 https://github.com/shirou/gopsutil

| ItemKey       | 说明                            | 单位 |
| :------------ | :------------------------------ | :--: |
| system.info   | 操作系统信息                    |      |
| agent.go.ping | 类似 agent.ping 检查 agent 状态 |      |
| cpu.model     | cpu 类型                        |      |
| mem.total     | 总内存                          |  MB  |
| mem.used      | 已使用内存                      |  MB  |
| mem.usedper   | 内存使用率                      |  %   |

### 源码编译

如需修改或编译生成二进制，可使用以下方法
golang 版本 1.11 以上

```
git clone https://github.com/canghai908/agent-go.git
cd agent-go/src/
export GO111MODULE=on
export GOPROXY=https://goproxy.io
go build
```

即可编译成功

### 使用

目前只提供 centos7_x64 位 rpm 包，下载并安装 RPM 包

```
wget https://dl.cactifans.com/tools/agent-go-0.0.1-1.el7.x86_64.rpm
rpm -ivh agent-go-0.0.1-1.el7.x86_64.rpm
```

启动 agent

```
systemctl start agent-go
```

查看 Agent 进程

```
ps -ef|grep agent-go
```

可以看到 agent 已经启动

```
root      5018     1  1 10:12 ?        00:00:00 /usr/bin/agent-go -c /etc/agent-go/agent-go.ini
```

Agent 配置文件位于

```
/etc/agent-go/agent-go.ini
```

配置文件介绍

> Debug 日志级别 1-7
> ListenIP agent 监听的 ip，默认为 0.0.0.0 所有 ip
> ListenPort agent 监听的 port，默认为 10049
> LogFile 日志路径，默认为/var/log/agent-go.log

Zabbix 中导入模版，新建一个主机，并关联导入的模版，主机名可以随意，IP 为部署 agent 的机器 ip，端口为 10049
![1](https://img.cactifans.com/wp-content/uploads/2019/03/4D5453E7-EC95-48C8-9C91-5DE47BD597AE-1024x430.jpg)
添加并关联模版。
![2](https://img.cactifans.com/wp-content/uploads/2019/03/78E7972B-23C0-44CC-96F3-56A1E4DD7F5F.jpg)
不出意外一会就可以看到主机的 Agent 状态已变为可用状态
![3](https://img.cactifans.com/wp-content/uploads/2019/03/B5DC034F-2378-4ABC-9A68-8746FB4853A8-1024x209.jpg)
并能在 Last data 里看到最新的数据
![4](https://img.cactifans.com/wp-content/uploads/2019/03/DE174094-827E-4F59-A4C4-4C7CCD8F5733-1024x420.jpg)

### Debug

如果想看到 Agent 返回的数据及交互，可以将配置文件里的 Debug 配置为最高模式 7，使用

```
systemctl restart agent-go
```

重启 agent，使用

```
tail -f /var/log/agent-go.log
```

即可看到数据的交互

```
2019/03/29 10:30:09.049 [D]  key: mem.total =======> 7982
2019/03/29 10:30:09.246 [D]  key: mem.used =======> 3854
2019/03/29 10:30:10.553 [D]  key: mem.usedper =======> 4.254296006902387E+01
2019/03/29 10:30:24.540 [D]  key: agent.go.ping =======> 1
2019/03/29 10:30:29.291 [D]  key: agent.go.ping =======> 1
2019/03/29 10:30:30.292 [D]  key: cpu.model =======> Intel(R) Xeon(R) CPU           L5520  @ 2.27GHz
2019/03/29 10:30:31.292 [D]  key: mem.total =======> 7982
2019/03/29 10:30:31.317 [D]  key: cpu.model =======> Intel(R) Xeon(R) CPU           L5520  @ 2.27GHz
2019/03/29 10:30:32.293 [D]  key: mem.used =======> 3854
2019/03/29 10:30:33.294 [D]  key: mem.usedper =======> 4.253503191571101E+01
2019/03/29 10:30:34.294 [D]  key: system.info =======> centos 7.6.1810
2019/03/29 10:30:35.295 [D]  key: agent.go.ping =======> 1
2019/03/29 10:30:36.297 [D]  key: cpu.model =======> Intel(R) Xeon(R) CPU           L5520  @ 2.27GHz
2019/03/29 10:30:37.297 [D]  key: mem.total =======> 7982
2019/03/29 10:30:38.297 [D]  key: mem.used =======> 3854
2019/03/29 10:30:39.298 [D]  key: mem.usedper =======> 4.2535080854928985E+01
2019/03/29 10:30:40.299 [D]  key: system.info =======> centos 7.6.1810
2019/03/29 10:30:40.800 [D]  key: system.info =======> centos 7.6.1810
```

### 压力测试

使用https://github.com/monitoringartist/zabbix-agent-stress-test 对 agent 进行压力测试，使用参数如下

```
./zabbix-agent-stress-test.py -s 127.0.0.1 -k "agent.go.ping" -t 20
```

压测结果
![5](https://img.cactifans.com/wp-content/uploads/2019/03/B9F9420E-0F04-424B-B541-4B349AA10394-1024x719.jpg)
agent 内存及 cpu 使用
![6](https://img.cactifans.com/wp-content/uploads/2019/03/87040134-9EA3-4CF0-A2CE-FFD6FC4BD3E2-1024x634.jpg)

## 总结

Zabbix 提供完全开源的源码，因此如 zabbix 自带功能无法实现，可以通过其协议实现自定义监控。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
