---
title: Zabbix Agent主动被动模式配置-1
date: 2019-03-22 16:04:49
tags: [zabbix, 被动模式, 主动模式]
categories: [zabbix]
---

长久以来对于安装 Zabbix Agent,文章介绍基本都是需要修改一下几个地方

```bash
Server=172.16.66.20
ServerActive=172.16.66.20
Hostname=node201
```

启动 Zabbix Agent 即可监控。对于具体为什么是这样配置，这样配置是主动模式还是被动模式？很少提起。本文主要介绍 Zabbxi Agent 的几个关键配置。

## 主动模式 VS 被动模式

![1](https://img.cactifans.com/wp-content/uploads/2019/03/7EBBC8A6-B2C2-40AE-B86F-55EEF378A02B.jpg)
Zabbix Agent 这二种模式，用通俗易懂的讲:
主动模式：Zabbix Agent 启动之后，把采集的数据主动发给 Zabbix Server 或者 Zabbix Porxy。
被动模式：Zabbix Server 或者 Zabbix Proxy 被动找 Zabbix Agent 拿监控数据。

![2](https://img.cactifans.com/wp-content/uploads/2019/03/F8C8B1D3-C40E-478E-BC90-188FD7BA2488.jpg)
这二种模式在使用过程中有所不同，各有优势，主要有以下区别

| 模式 | Server 压力 | 远程命令 | 日志监控 |
| :--: | :---------: | :------: | :------: |
| 主动 |     低      |  不支持  |   支持   |
| 被动 |     高      |   支持   |  不支持  |

鉴于以上不同，根据实际需求，选择对应模式。

## 主动模式配置

要配置主动模式，只需要配置以下几个参数即可

```bash
ServerActive=172.16.66.20
Hostname=node201
```

配置解释：
**ServerActive** 配置为 Zabbix Server 或 Zabbix proxy 的地址，这里可以配置域名/ip，如需配置多个地址，多地址之间用英文逗号隔开即可，如：192.168.1.100,10.10.1.100
**Hostname** 配置唯一的主机名，以便识别此机器。在 Zabbix 里，不同主机的区分就是通过 hostname 区分的，并不是通过 IP！！！因此这里建议进行规划，按照一定规律配置，比如区域-机房-业务-ip 等形式配置，此配置建议使用英文（有人修改只后可使用中文的），也可按照 FQDN 规则配置主机名。FQDN https://en.wikipedia.org/wiki/Fully_qualified_domain_name
Zabbix Agent 配置这 2 个参数之后，即可使用主动模式，如果要想非常“纯粹”的使用主动模式，而关闭被动模式，还需要修改一个配置

```
StartAgents=0
```

在 Agent 中对于此配置有详细介绍（很多人都不看...）

> Number of pre-forked instances of zabbix_agentd that process passive checks.If set to 0, disables passive checks and the agent will not listen on any TCP port.

此项目配置被动模式下 zabbix agentd 所派生的进程数量。如果配置为 0，会关闭被动模式检查，而且 Agent 不会监听任何主机 TCP 端口!!!。因为是数据是从 Agent 发出，因此主机放行对外通行即可(一般都放开)，不需要在主机上添加配置防火墙规则。查看进程如下
![3](https://img.cactifans.com/wp-content/uploads/2019/03/DCFC9524-E680-41E5-B1AF-FC2122AD51F6.jpg)

## 主机配置

主动模式配置之后，Agent 启动之后就会发送监控到 Server 或者 Proxy，如找不到对应的主机名，agent 日志会有如下报错

```
no active checks on server [172.16.66.20:10051]: host [node201] not found
```

Zabbix Server 或 Zabbix Proxy 的日志也会看到如下报错

```
cannot send list of active checks to "172.16.66.20": host [node201] not found
```

此时就需要通过手动添加或 Zabbix Server 的自动注册功能，将主机添加到 Zabbix Server。
如果为手动添加主机，需要在 Zabbix Server 添加主机
![5](https://img.cactifans.com/wp-content/uploads/2019/03/E7B8C74F-7072-4DAD-A2D5-41662C7B3562.jpg)
**Host name** 为必须配置项目，需要和 Agent 配置里的 Hostname 配置一致！！！
**Visible name** 配置为可见名称，这里可配置为中文，主机列表会显示此名称
**Agent interfaces** 的 IP 和端口可以随意配置！！！不过还是建议配置成业务 ip 或者主机的真实 IP。

## 模版配置

由于禁用的 Agent 的被动模式，而 Zabbix Serve 自带的很多模版采集指标基本为**被动模式**，因此需要将模式改为被动。建议克隆原模版之后，将新模版监控指标类型修改为 Zabbix Active 模式即可正常采集数据。
![6](https://img.cactifans.com/wp-content/uploads/2019/03/B4FE905D-EB85-4762-AF93-F8CBAFA9C1A3.jpg)

## 注意事项

如纯使用主动模式，需要注意以下适宜 1.主动模式不支持远程命令执行。如你需要在 Zabbix Agent 执行远程命令，需要 Agen 开启被动模式。
2.Agent 自带的日志监控，仅支持主动模式，不支持被动模式。 3.主动模式建议为指标配置 nodate 告警阈值。 4.利用主动模式，可将 Zabbix Server 或者 Zabbix Proxy 放在公网，内网 Zabbix Agent 配置主动模式，即可监控内网机器。

下期内容介绍：Zabbix Agent 主动被动模式配置-2 被动模式配置及使用

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
