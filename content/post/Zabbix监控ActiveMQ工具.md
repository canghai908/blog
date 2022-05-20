---
title: Zabbix监控ActiveMQ工具
date: 2018-08-20 16:41:04
tags: [zabbix, activemq, LLD]
categories: [zabbix]
---

## 介绍

最近学习使用 go 语言写了一个 zabbix 监控 ActiveMQ 的小工具，有如下特点: 1.使用 Zabbix Agent Trapper 方式(主动发送采集数据到 zabbix server，类似 active 模式)监控 Activemq 状态 2.支持对密码加密，避免配置文件里出现明文密码 3.支持 ActiveMQ 基本状态/Queues/Topics 状态监控 4.支持自定义采集周期
5.LLD（自动发现）添加 Queues/Topics 状态监控
工具通过访问 ActimveMQ 管理页面，登录之后抓取页面数据，进行采集

模版下载:[https://dl.cactifans.com/zabbix/zabbix_template_activemq.tar.gz](https://dl.cactifans.com/zabbix/zabbix_template_activemq.tar.gz)
脚本下载:[https://dl.cactifans.com/zabbix/mqmon-0.0.1_linux_amd64.tar.gz](https://dl.cactifans.com/zabbix/mqmon-0.0.1_linux_amd64.tar.gz)

## 导入模版

在 zabbix Server 上导入导入模版，解压之前下载的模版。
导入模版文件：zbx_templates_activemq.xml
![2](https://img.cactifans.com/wp-content/uploads/2018/08/2.jpg)
导入之后可以看到名为 Template App ActiveMQ 的模版，表示导入成功
![3](https://img.cactifans.com/wp-content/uploads/2018/08/4-1.jpg)
**Activemq 作为中间件可以挂载到任何在 zabbix server 里的 host 上！！！\*\***监控脚本不一定部署在真实的 Activemq 服务器之上\*\*，只要脚本通过远程方式能连接到 activemq 管理页面即可。关联模版到需要挂载 ActiveMQ 监控的的 host 上即可。

## 配置插件

下载并解压插件

```
mkdir -p  /opt/mqmon
wget https://dl.cactifans.com/zabbix/mqmon-0.0.1_linux_amd64.tar.gz
tar zxvf mqmon-0.0.1_linux_amd64.tar.gz -C /opt/mymon
```

文件目录结构
├── control //启动脚本
├── mqmon //二进制程序
└── mqmon.json //配置文件
把密码写在明文的文件里不被推荐的，因此脚本提供了一个使用 AES 加密算法加密管理员密码的工具，保证管理员密码的安全。使用以下命令加密密码明文，将 yourpassword 替换为你的密码

```bash
/opt/mqmon/mqmon enc yourpassword
```

执行之后会看到进过加密后的密码密文。记录下来

```bash
/opt/mqmon/mqmon enc admin
sXcEQ2FTGk4WsWSxyT6fuBnjZ3v43pc0
```

修改配置文件 mqmon.json

```
{
"debug": true,
"interval":{
	"status": 300,
	"discovery": 300,
	"metic": 60
},
"activemq": {
    "username": "admin",
    "password": "j1wqc+QGX2+7n/KOlEmNPZQsaWhmkqGQ",
    "host": "172.16.66.16",
    "port": 8161
    },
"zabbix":{
	"server": "zabbix.cactifans.com",
	"port": 10051,
	"hostname": "host135"
    }
}

```

配置文件说明
interval 采集周期配置，单位为秒

> status ativemq 基本信息采集周期，默认为 300 秒
> discovery queues 和 topics 自动发现周期，默认为 300 秒
> metic queues 和 topics 具体指标采集周期，默认为 60 秒

需要监控的 Activemq 信息配置

> username activemq 管理账号
> passoword activemq 用户密码加密后的密文  
> host activemq 的主机  
> port activemq 管理页面端口,默认为 8161

zabbix 信息配置

> server 为 zabbix server 的地址，如通过 zabbix proxy 需要设置为 zabbix proxy 的地址  
> port zabbix server 端口默认为 10051  
> hostname 为之前关联模版的主机名一致

![4](https://img.cactifans.com/wp-content/uploads/2018/08/4.jpg)

## 使用

配置需要监控的 activemq 的管理页面地址/端口/用户账号/密码信息之后，
可以启动插件，使用以下命令进行测试 activemq 是否能够连通

```
cd /opt/mqmon
./mqmon ping
```

可以看到使用的配置文件，如返回 OK，表示配置信息正确，如其他表示连接异常，请检查配置文件及网络。

```bash
2018/08/20 12:58:44 ping.go:37: Using config file: /opt/mqmon.json  successfully!
OK
```

测试成功之后可以使用以下命令启动即可

```bash
./control start
```

常用操作

```
./control start    //启动应用
./control stop     //停止应用
./control restart  //重启应用
./control tail     //查看日志
```

## 效果

![5](https://img.cactifans.com/wp-content/uploads/2018/08/1-1.jpg)
![6](https://img.cactifans.com/wp-content/uploads/2018/08/2-1.jpg)
![7](https://img.cactifans.com/wp-content/uploads/2018/08/3-1.jpg)
![8](https://img.cactifans.com/wp-content/uploads/2018/08/5-1.jpg)

## 扩展

### 自动发现数据结构

Queue 自动发现规则数据（activemq.queues.discovery 的数据）Demo

```
{
	"data": [
	{"{#QUEUENAME}": "demoRequest"},
	{"{#QUEUENAME}": "demoResponse"}
	]
}
```

Topic 自动发现规则数据（activemq.topics.discovery 的数据）Demo

```
{
	"data": [
	    {"{#TOPICNAME}": "12312312"},
		{"{#TOPICNAME}": "12312"},
		{"{#TOPICNAME}": "23121"}
		]
}
```

### 添加阈值

如需要对特定的 Queue 或 topic 添加阈值,可以根据以下方式配置
例子：假如要为 demoRequest 和 demoResponse 2 个 Queue 添加二个阈值，规则如下
1.demoRequest 的 Number Of Pending Messages 大于 10 告警
2.demoResponse 的 Number Of Pending Messages 大于 5 告警 3.其他 queue 大于 2 告警
可进行如下配置：
在模版里添加如下规则
![6](https://img.cactifans.com/wp-content/uploads/2018/08/6-1.jpg)
注意是在模版 Discovery rules 里的 Trigger prototypes 里添加如下触发器原型
标题

```
Queue {#QUEUENAME} PM > {$QUEUE_PM_LIMIT:"{#QUEUENAME}"}
```

表达式

```
{Template App ActiveMQ:activemq.queues.pm[{#QUEUENAME}].last()}>{$QUEUE_PM_LIMIT:"{#QUEUENAME}"}
```

创建之后，在关联的 hosts 上配置如下宏。宏可以配置在模版里，为了灵活性需要，建议配置到具体的主机上，可做到精确匹配

```
{$QUEUE_PM_LIMIT:demoRequest}     =>  10
{$QUEUE_PM_LIMIT:demoResponse}    =>  5
{$QUEUE_PM_LIMIT}                 =>  2
```

如图
![10](https://img.cactifans.com/wp-content/uploads/2018/08/9-1.jpg)
即可完成为不同 queue 配置不同的阈值的需求。

```
{#QUEUENAME}为Queue自动发现的数据，topic设置为{#TOPICNAME}
{$QUEUE_PM_LIMIT} 为自定义的宏，可以随意起名
```

![11](https://img.cactifans.com/wp-content/uploads/2018/08/10.jpg)
关于具体的说明，可以看官方这里的列子:
[https://www.zabbix.com/documentation/3.4/manual/discovery/low_level_discovery](https://www.zabbix.com/documentation/3.4/manual/discovery/low_level_discovery)

### 命令行工具

工具内置几个命令行工具及基本使用,可以使用 mqmon -h 查看帮助

```bash
[root@localhost activemq-mymon]# ./mqmon
Zabbix activemq monitoring tool. For example:

mqmon daemon --config=./mqmon.json

Usage:
  mqmon [command]

Available Commands:
  daemon      Running as a daemon
  enc         Encrypt passwords in AES mode
  help        Help about any command
  ping        Connected line checker
  version     Version

Flags:
      --config string   config file (default is /.mqmon.json)
  -h, --help            help for mqmon

Use "mqmon [command] --help" for more information about a command.
```

## 注意事项

1.trapper 方式默认允许任何主机发送数据到 zabbix server，建议通过设置宏的方式，在模版里配置 allowed hosts 配置权限 2.采集指标建议采用默认采集周期，缩短采集周期会对 activemq 页面产生一定压力

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
