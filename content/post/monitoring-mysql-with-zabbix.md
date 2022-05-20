---
title: "monitoring mysql with zabbix"
date: 2015-07-29 23:38:22
tags:
  - zabbix
  - mysql
---

使用 go 语言写了一个采集 mysql 性能的小程序，通过 SDK 连接 mysql 数据库，采集数据库性能指标，同时采集远程数据库的性能，大家可试用一下，欢迎提出修改意见和建议

## 被监控端设置

监控下载：
Linux：
[mymon_x86.tar.gz](https://dl.cactifans.com/tools/mymon_x86.tar.gz)
[mymon_x64.tar.gz](https://dl.cactifans.com/tools/mymon_x64.tar.gz)

Linux:
修改 zabbix agentd 配置文件(具体位置根据自身情况设置)，添加 key

```
vi /usr/local/zabbix/etc/zabbix_agentd.conf
```

添加如下

```
#mysql
UserParameter=mysql.status[*],/usr/local/zabbix/bin/mysql/mymon $1 $2
```

添加好之后执行

```
mkdir -p /usr/local/zabbix/bin/mysql/
cd /usr/local/zabbix/bin/mysql/
wget https://dl.cactifans.com/tools/mymon_x64.tar.gz
tar zxvf mymon_x64.tar.gz
chmod a+x mymon
```

同时编辑 cfg.json 配置文件，设置要监控数据库的连接参数

```
vi /usr/local/zabbix/bin/mysql/cfg.json
```

文件内容如下

```
{
"mysql": {
        "username": "root",
        "password": "123456",
        "host": "localhost",
        "port": 3306
    },
"sever": {
                "log": "/tmp/mon.log"
        }
}
```

登录数据库的账号，密码等信息，log 项目不用设置，默认即可
设置好之后重启 zabixx agent

```
service zabbix-agent restart
```

## zabbix 服务端设置

测试 agent 是否工作正常

```
/usr/local/bin/zabbix_get -s 192.168.7.186 -k mysql.status[GlobalStatus,Uptime]
```

如果有一下回显，表示工作正常

```
[root@vm153 ~]#  /usr/local/bin/zabbix_get -s 192.168.7.186 -k mysql.status[GlobalStatus,Uptime]
12364726
[root@vm153 ~]#
```

导入模版：[zabbix_mysql_templates.xml](https://dl.cactifans.com/tools/zabbix_mysql_templates.tar.gz)
并关联到主机
效果
![screen1](https://img.cactifans.com/wp-content/uploads/2015/07/1.png)
参数
![screen2](https://img.cactifans.com/wp-content/uploads/2015/07/2.png)

PS：可自行添加对应 key 可监控项目，模版自带了几种，脚本包括 800 多项目，
![screen3](https://img.cactifans.com/wp-content/uploads/2015/07/3.png)
具体更具 key 定义，括号内第一项为 key 类型，分 GlobalStatus 和 GlobalVariables 二种，后面的数值为在 mysql 里执行“SHOW /_!50001 GLOBAL _/ STATUS”和“SHOW /_!50001 GLOBAL _/ VARIABLES”，对应的 key，注意有些数值为固定数值，有些 key 的类型为每秒速差，自行添加.
