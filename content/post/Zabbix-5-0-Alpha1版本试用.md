---
title: Zabbix 5.0 Alpha1版本试用
date: 2020-03-31 13:51:00
tags: [5.0, redis]
categories: [zabbix]
---

Zabbix 5.0 Alpha1 于 2020.年 1 月 20 日发布。作为 5.0 的的第一个版本，可抢先体验 5.0 的部分功能。

## 安装

本次使用源码编译进行安装，与之前版本安装方法变化不大

### Zabbix Server

下载后解压。Zabbix 5.0 可使用 agent2,因此使用以下参数编译安装

```bash
./configure --prefix=/usr/local/zabbix \
--enable-server \
--enable-agent2 --with-mysql \
--with-net-snmp --with-libcurl \
--with-libxml2
```

配置

![1](https://img.cactifans.com/wp-content/uploads/2020/02/2020-02-18_102628.png)
安装之后，修改 zabbix_server.conf 的数据库相关信息，启动 zabbix server 即可。

```bash
/usr/local/zabbix/sbin/zabbix_server -c /usr/local/zabbix/etc/zabbix_server.conf
```

使用以下命令，启动 agent2

```bash
/usr/local/zabbix/sbin/zabbix_agent2 -c /usr/local/zabbix/etc/zabbix_agent2.conf
```

### Zabbix web

**PHP 版本要求 php7.2 版本**，低于此版本，web 登录时会显示 http 500 错误。
![2](https://img.cactifans.com/wp-content/uploads/2020/02/2020-02-18_102955.png)
按照步骤，初始化完成即可。
![3](https://img.cactifans.com/wp-content/uploads/2020/02/2020-02-18_141602.png)

## Zabbix Agent2 监控 Redis

新版本的 agent2，内置了 redis 监控,可直接使用。并有如下特点

- 支持 tcp 和 socket 连接 redis
- 支持带密码的 redis 监控
- 支持多个 redis 监控
- 支持 key 监控
- .....

安装 redis

```bash
yum install -y redis
```

启动 redis

```
systemctl enable --now redis
```

相关配置文件如下
![4](https://img.cactifans.com/wp-content/uploads/2020/02/2020-02-18_171023.png)
修改 zabbix_agent2.conf 配置文件,以下地方

```
Plugins.Redis.Uri=tcp://localhost:6379           #支持tcp连接和socket连接
Plugins.Redis.Password=                          #redis 密码
```

配置之后，关联 zabbix 自带的 redis 模板即可。
效果
![5](https://img.cactifans.com/wp-content/uploads/2020/02/2020-02-18_155441.png)

## 总结

Zabbix 5.0 版本计划 2020 年 3 月发布，Agent2 使用 golang 语言编写，可方便编写各种插件，灵活配置监控，期待中！
