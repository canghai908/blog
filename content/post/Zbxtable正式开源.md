---
title: ZbxTable正式开源
tags: [zabbix, 报表]
categories: [zbxtable]
date: 2020-07-19T15:11:56+08:00
---

## 系统介绍

ZbxTable 是使用 Go 语言开发的一个开源的 Zabbix 报表系统。基本功能如下：

- 导出监控指标特定时间段内的详情数据与趋势数据到 xlsx
- 导出特定时间段内 Zabbix 的告警消息到 xlsx
- 对特定时间段研内的告警消息进行分析，告警 Top10 等
- 按照主机组导出巡检报告
- 对 Zabbix 图形按照数类型进行显示和查看并支持导出到 pdf
- 主机未恢复告警显示和查询

## 系统架构

![1](https://img.cactifans.com/wp-content/uploads/2020/07/zbxtable.png)

## 组件介绍

ZbxTable: 使用 beego 框架编写的后端程序

ZbxTable-Web: 使用 React 编写的前端

MS-Agent: 安装在 Zabbix Server 上,用于接收 Zabbix Server 产生的告警，并发送到 ZbxTable 平台

## 在线体验

直接点击登录即可

[https://zbx.cactifans.com](https://zbx.cactifans.com)

## 兼容性

| zabbix 版本 | 兼容性            |
| :---------- | :---------------- |
| 5.0.x LTS   | ✅                |
| 4.4.x       | ✅                |
| 4.2.x       | ✅                |
| 4.0.x LTS   | ✅                |
| 3.4.x       | ❓ 理论支持未测试 |
| 3.2.x       | ❓ 理论支持未测试 |
| 3.0.x LTS   | ❓ 理论支持未测试 |

## 源码及 RPM 包

## 1.源码

ZbxTable: [https://github.com/canghai908/zbxtable](https://github.com/canghai908/zbxtable)

ZbxTable-Web: [https://github.com/canghai908/zbxtable-web](https://github.com/canghai908/zbxtable-web)

MS-Agent: [https://github.com/canghai908/ms-agent](https://github.com/canghai908/ms-agent)

## 2.RPM 包：

ZbxTable: [https://dl.cactifans.com/zabbix/zbxtable-1.0.0-1.el7.x86_64.rpm](https://dl.cactifans.com/zabbix/zbxtable-1.0.0-1.el7.x86_64.rpm)

ZbxTable-Web: [https://dl.cactifans.com/zabbix/zbxtable-web-1.0.0-1.el7.x86_64.rpm](https://dl.cactifans.com/zabbix/zbxtable-web-1.0.0-1.el7.x86_64.rpm)

MS-Agent: [https://dl.cactifans.com/zabbix/ms-agent-1.0.0-1.el7.x86_64.rpm](https://dl.cactifans.com/zabbix/ms-agent-1.0.0-1.el7.x86_64.rpm)

系统默认账号：admin 密码：Zbxtable

## 安装部署

系统采用前后端分离，可与 zabbix 安装到一台服务器，也可分开部署。部署步骤如下

## 1.后端部署

环境部署要求：
操作系统：centos7 x64

数据库：MySQL

## 1.1 创建数据库用户

```
# mysql -uroot -p
password
mysql> create database zbxtable character set utf8 collate utf8_bin;
mysql> create user zbxtable@localhost identified by 'zbxtablepwd123';
mysql> grant all privileges on zbxtable.* to zbxtable@localhost;
mysql> quit;
```

## 1.2 后端安装

```
yum install https://dl.cactifans.com/zabbix/zbxtable-1.0.0-1.el7.x86_64.rpm -y
```

## 1.3 修改配置文件

配置文件在 conf/app.conf

```
#zbxtable
appname = zbxtable
httpport = 8084
runmode = prod
autorender = false
copyrequestbody = true
EnableDocs = true
appurl = http://192.168.10.10:8088
#session过期时间，单位为小时，默认12小时。如需大屏自动刷新，建议配置较大配置时间
session_timeout = 12

#database
hostname = localhost
username = zbxtable
dbpsword = zbxtablepwd123
database = zbxtable
port = 3306
dbprefix = zbxtable_

#zabbix server info
zabbix_server = http://192.168.10.12
zabbix_user = admin
zabbix_pass = zabbix
#alarm send token
token = ec573cf7388da56916f75ba9bbe46a69
```

主要配置有以下

- appurl = http://ip:8088 为最终系统对外的访问地址，与图形显示有关
- zabbix server info 为 zabbix server 的地址及账号密码
- token 为 ms-agent 与 ZbxTable 平台通信的 token，可自行修改,与 ms-agent 配置的 token 保持一致即可，具体可查看 ms-agent 文档https://github.com/canghai908/ms-agent

## 1.4 启动

修改好配置后，使用以下命令启动

```
systemctl enable --now zbxtable
```

重启

```
systemctl restart zbxtable
```

## 1.5 Debug

如启动失败或者出现错误错误，可修改通过需改程序配置文件,修改运行模式为 dev 模式,并重启 zbxtable,查看程序日志解决,日志位于 logs/zbxtable.log

## 2 前端部署

安装

```
yum install https://dl.cactifans.com/zabbix/zbxtable-web-1.0.0-1.el7.x86_64.rpm -y
```

安装好之后文件位于/usr/local/zbxtable/web
前端为纯静态文件，需使用 nginx，如机器未安装 nginx，使用以下命令安装 nginx

```
yum install nginx -y
```

拷贝 nginx 配置文件

```
cp /usr/local/zbxtable/nginx.conf /etc/nginx/conf.d/
```

重启 nginx

```
systemctl restart nginx
```

配置开机启动

```
systemctl enable  nginx
```

使用http://ip:8088 即可访问系统，系统默认账号：admin 密码：Zbxtable

## 3 ms-agent 部署

ms-agent 必须部署在 Zabbix Server 服务器,ms-agent 接收 zabbix 的告警消息，通过 http 协议发送到 ZbxTable 平台

## 3.1 配置

ms-agent 需使用 zbxtable 命令完成在 Zabbix Server 的配置,包括创建用户,配置动作等配置。配置过程如下,确保 ZbxTable 配置文件里的 Zabbix Server 信息配置正确

```
cd /usr/local/zbxtable
./zbxtable install
```

显示如下日志

```
2020/07/18 23:22:16.881 [I] [install.go:43]  Zabbix API Address: http://zabbix-server/api_jsonrpc.php
2020/07/18 23:22:16.881 [I] [install.go:44]  Zabbix Admin User: Admin
2020/07/18 23:22:16.881 [I] [install.go:45]  Zabbix Admin Password: xxxxx
2020/07/18 23:22:17.716 [I] [install.go:52]  登录zabbix平台成功!
2020/07/18 23:22:17.879 [I] [install.go:69]  创建告警媒介成功!
2020/07/18 23:22:18.027 [I] [install.go:82]  创建告警用户组成功!
2020/07/18 23:22:18.198 [I] [install.go:113]  创建告警用户成功!
2020/07/18 23:22:18.198 [I] [install.go:114]  用户名:ms-agent
2020/07/18 23:22:18.198 [I] [install.go:115]  密码:xxxx
2020/07/18 23:22:18.366 [I] [install.go:167]  创建告警动作成功!
2020/07/18 23:22:18.366 [I] [install.go:168]  插件安装完成!
```

表示配置成功.此步骤会在 Zabbix Server 创建 ms-agent，密码为随机，并配置相关 Action 和 Media Type，并关联到用户.

## 3.2 安装

此程序必须部署在 Zabbix Server

```
yum install https://dl.cactifans.com/zabbix/ms-agent-1.0.0-1.el7.x86_64.rpm -y
```

环境信息

| 程序     | 路径                                  | 作用                                             |
| :------- | :------------------------------------ | :----------------------------------------------- |
| ms-agent | /usr/lib/zabbix/alertscripts/ms-agent | 接收 Zabbix 平台产生的告警并发送到 ZbxTable 平台 |
| app.ini  | /etc/ms-agent/app.ini                 | ms-agent 配置文件                                |

如果你的 Zabbix Server 的 alertscripts 目录不为/usr/lib/zabbix/alertscripts/ 需要移动 ms-agen 到你的 zabbix server 的 alertscripts 目录下即可,否则会在 Zabbix 告警页面出现找不到 ms-agent 的错误提示，也无法收到告警消息。
也可以修改 Zabbix Server 的配置文件，将 alertscripts 目录指向/usr/lib/zabbix/alertscripts/

vi zabbix_server.conf

```
AlertScriptsPath=/usr/lib/zabbix/alertscripts
```

修改后重启 Zabbix Server 生效

## 3.3 配置文件

/etc/ms-agent/app.ini 为程序配置文件，默认内容如下

```
[app]
Debug = 1
LogSavePath = /tmp
Host = http://192.168.10.10:8088/v1/receive
Token = ec573cf7388da56916f75ba9bbe46a69
```

Debug 为程序日志级别 0 是 debug，1 为 info

LogSavePath 为日志目录，默认为/tmp 目录

Host 为 ZbxTable 系统地址，默认为 http 服务器 IP+/v1/receive

Token 与 ZbxTable 通信的 Token,可自行修改,需要与 ZbxTable 平台配置保持一致即可，否则无法接收告警。

## 3.4 Debug

可修改配置文件打开 Debug 模式，查看日志文件名格式如下/tmp/ms-agent_yyyymmdd.log

## Team

后端

[canghai908](https://github.com/canghai908)

前端

[ahyiru](https://github.com/ahyiru)

## 系统截图

系统登录
![1](https://img.cactifans.com/wp-content/uploads/2020/07/C893A6D6-D4BC-4E0F-A087-2826E6D12699.png)
系统首页
![1](https://img.cactifans.com/wp-content/uploads/2020/07/BA043817-6994-44FE-8FC2-36D2561C0C92.png)
主机列表
![1](https://img.cactifans.com/wp-content/uploads/2020/07/7F477F6C-8BB0-4042-A643-0D3902C706E3.png)
图形管理
![1](https://img.cactifans.com/wp-content/uploads/2020/07/6F1BD4F7-B185-4D49-A48C-4E9E0E6900A4.png)
图形导出
![1](https://img.cactifans.com/wp-content/uploads/2020/07/2DE17FD9-320B-4685-BCEF-08BFA834A8F7.png)
指标导出
![1](https://img.cactifans.com/wp-content/uploads/2020/07/99E941CE-0018-4AB3-A8B0-35795FA852FA.png)
巡检报告导出
![1](https://img.cactifans.com/wp-content/uploads/2020/07/88B8D652-183A-4595-ACFD-47157FD230FB.png)
告警分析
![1](https://img.cactifans.com/wp-content/uploads/2020/07/482304D1-FD67-4C09-B6E5-7D0660D69556.png)
告警导出
![1](https://img.cactifans.com/wp-content/uploads/2020/07/D2C82FC2-D6F2-472C-BA32-F8825A12F7CA.png)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
