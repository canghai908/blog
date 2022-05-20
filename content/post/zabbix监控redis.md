---
title: zabbix 监控 redis
date: 2018-01-20 16:16:20
tags: [zabbix, redis]
categories: [zabbix]
---

使用 go 语言写了一个采集 redis 性能的小程序，会自动发现机器上安装的所有 redis，并通过 go 客户端连接 redis，采集 redis 性能指标，进行监控。欢迎提出修改意见和建议。
使用注意点：

- 目前不支持设置密码的 redis
- 注意设置 sudo 权限

本次使用的 zabbix 环境，其他 zabbix 版本也支持

| 环境               |       版本 |
| :----------------- | ---------: |
| zabbix server 版本 |      2.4.4 |
| zabbix agent 版本  |      2.4.4 |
| centos             | 6.6 x86_64 |

# 客户端配置

## 客户端下载

linux 32 位系统
[zabbix_redis.x86.tar.gz](https://dl.cactifans.com/tools/zabbix_redis.x86.tar.gz)
linux64 位系统
[zabbix_redis.x86_64.tar.gz](https://dl.cactifans.com/tools/zabbix_redis.x86_64.tar.gz)

## 修改配置

修改 zabbix agentd 配置文件(具体位置根据自身情况设置)，添加 key
添加如下内容

```
UserParameter=redis.port.discovery,sudo /usr/local/zabbix/bin/redis/redis_discovery
UserParameter=redis[*],/usr/local/zabbix/bin/redis/redis $1 $2
```

添加好之后执行(zabbix-agent 安装路径为/usr/local/zabbix/)

```
cd /usr/local/zabbix/bin/
wget https://dl.cactifans.com/tools/zabbix_redis.x86_64.tar.gz
tar zxvf zabbix_redis.x86_64.tar.gz
```

添加之后，需要重启 zabbix agent，由于需要 sudo 权限，因此需要修改 sudoer 文件,

```
vi /etc/sudoers
```

添加如下内容

```
zabbix  ALL=NOPASSWD:/usr/local/zabbix/bin/redis/redis_discovery
Defaults:zabbix  !requiretty
```

## 测试执行

```
/usr/local/zabbix/bin/redis/redis_discovery
```

执行之后,可显示本机所有 redis 端口（json 格式）

```
{"data":[{"{#PORT}":"6379"},{"{#PORT}":"6380"}]}
```

表示配置成功

# server 端配置

下载并导入 redis 监控模版: [zabbix_redis_templates.tar.gz](https://dl.cactifans.com/tools/zabbix_redis_templates.tar.gz)
关联 redis 模版，即可查看数据
监控效果

![1](https://img.cactifans.com/wp-content/uploads/2015/11/1.png)

![2](https://img.cactifans.com/wp-content/uploads/2015/11/2.png)

![3](https://img.cactifans.com/wp-content/uploads/2015/11/3.png)

![4](https://img.cactifans.com/wp-content/uploads/2015/11/4.png)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
