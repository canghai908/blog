---
title: Zabbix Agent主动被动模式配置-2
date: 2019-05-31 14:18:32
tags: [被动模式, 远程命令]
categories: [zabbix]
---

接上期，长久以来对于安装 Zabbix Agent,文章介绍基本都是需要修改一下几个地方

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

## 被动模式配置

要配置被动模式，只需要配置以下几个参数即可

```bash
Server=172.16.66.20
```

**Server** 配置为 Zabbix Server 或 Zabbix proxy 的地址，这里可以配置域名/ip，如需配置多个地址，多地址之间用英文逗号隔开即可，如：192.168.1.100,10.10.1.100。这里可以理解为 ACL 功能，即允许那些 Zabbix Server 及 Zabbix Proxy 访问 Zabbix Agent，因此可以配置网段和配置成所有 IP，如：192.168.1.0/24 或 0.0.0.0/0，纯被动模式下只需要配置 Server 即可 Hostname 不配置。
配置好之后重启 Zabbix Agent 查看进程如下
![3](https://img.cactifans.com/wp-content/uploads/2019/05/E9C45B5A-7E2A-4866-A7B5-F504E050CF8C-1024x97.jpg)

## 主机配置

被动模式配置之后，启动 zabbix agent，需要在 Zabbix Server 添加主机
![4](https://img.cactifans.com/wp-content/uploads/2019/05/3FB560C8-1E77-4CC9-8A87-B9CAB34EF7A5-1024x493.jpg)
**Host name** 可随意配置，建议按照一定规则，不与其他机器重复即可
**Visible name** 配置为可见名称，这里可配置为中文，主机列表会显示此名称
**Agent interfaces** agent 所在机器的 IP 和端口，这里一定要配置成 agent 真实的 IP，默认 zabbix agent 端口为 10050，可以通过配置文件进行修改

## 模版配置

zabbix 自带模版大多数为被动模式，因此直接关联模版即可进行监控。

## 远程命令

与主动模式不同，被动模式支持 zabbix agent 执行远程命令。在出现告警后，发送邮件的同时，可以配置远程命令实现故障“自愈”，如重启服务等

### Zabbix Agent 配置

在 Zabbix Agent 配置文件中开启

```bash
EnableRemoteCommands=1
LogRemoteCommands=1
```

**EnableRemoteCommands** 配置开启远程命令
**LogRemoteCommands** 在日志中记录远程命令
配置之后重启 Zabbix Agent

### 系统配置

由于 zabbix 执行命令默认是以**zabbix** 系统用户执行，因此需要在操作系统上为 zabbix 用户配置 sudo。
输入

```bash
visudo
```

添加

```bash
zabbix ALL=NOPASSWD: ALL
```

配置为 ALL，这里也可以配置成具体的命令

```bash
zabbix ALL=NOPASSWD: /etc/init.d/nginx restart
```

### Zabbix Server 配置

以下举例配置一个 Nginx 自动恢复，使用 zabbix agent 远程命令实现 nginx 停止自动启动。
配置如下 Item，监控 Nginx 的 80 端口
![5](https://img.cactifans.com/wp-content/uploads/2019/05/1C0C9A26-98D2-4FF6-8011-5ADC357B5628-1024x191.jpg)
并配置一个 Trigger 端口不存在时告警
![6](https://img.cactifans.com/wp-content/uploads/2019/05/BBDF18EA-20B1-44F7-940B-4318AF72CA9D-1024x29.jpg)
配置一个 Action，条件为 Trigger 名称，端口 down 之后，执行脚本重启 Nginx
![7](https://img.cactifans.com/wp-content/uploads/2019/05/2A6B87ED-2966-4DBF-A918-35C99F362C3B-1024x547.jpg)
为 Action 配置 Operations，添加一个远程执行命令
![8](https://img.cactifans.com/wp-content/uploads/2019/05/2CA520FA-732F-49C3-A0D2-8EA03F1CEB48-1024x619.jpg)
配置命令和执行主机
![9](https://img.cactifans.com/wp-content/uploads/2019/05/1EE3DC72-D559-466F-8003-F1DEAFC7D010-1024x712.jpg)
配置好之后，查看下目前 Nginx 的状态为启动状态
![10](https://img.cactifans.com/wp-content/uploads/2019/05/D2E6EBA3-3884-44A4-8C5F-0193A761C7EC-1024x266.jpg)
启动时间为 14:27
手动停止 Nginx 服务，一分钟之后在 Zabbix Problems 查看
![10](https://img.cactifans.com/wp-content/uploads/2019/05/54C76362-9D72-447B-8ED1-754A974C2934-1024x68.jpg)
看到 44 分故障 45 分已经恢复
查看历史数据
![11](https://img.cactifans.com/wp-content/uploads/2019/05/0A9F2AFB-E039-432C-9684-B4565DDBDC58.jpg)
可以看到 44 分 Nginx 端口为 down，45 分恢复正常
查看 Nginx 状态
![12](https://img.cactifans.com/wp-content/uploads/2019/05/EC2B3B1D-B542-4688-8B3B-9E5A6F799C84-1024x273.jpg)
Nginx 运行正常，启动时间为 44 分
查看 Zabbix Agent 日志
![13](https://img.cactifans.com/wp-content/uploads/2019/05/AC709984-AC0C-4436-8064-E9945F065D81.jpg)
可以看到 44 分执行了脚本,初步实现了简单故障的“自愈”。

## 注意事项

被动模式远程命令，是以 zabbix 用户执行，注意配置 sudo 及权限，命令后可跟参数，参数可以使用 zabbix 的内置宏。

点击下方【查看原文】到 Blog 阅读原文
