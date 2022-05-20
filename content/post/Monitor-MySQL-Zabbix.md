---
title: "Monitor MySQL Zabbix"
date: 2015-04-03 15:46:33
tags:
  - zabbix
  - mysql
categories: [zabbix]
---

环境

| 操作系统              |            CentOS7 |
| :-------------------- | -----------------: |
| Zabbix 版本           |              2.4.3 |
| MySql 版本            |             5.6.23 |
| zabbix_agent 安装目录 | /usr/local/zabbix/ |

# 被监控端配置

下载监控模版 m
[percona-monitoring-plugins-1.1.4.tar.gz](https://dl.cactifans.com/template/percona-monitoring-plugins-1.1.4.tar.gz)
下载到客户端/opt 目录下

```
cd /opt
wget https://dl.cactifans.com/template/percona-monitoring-plugins-1.1.4.tar.gz
tar zxvf percona-monitoring-plugins-1.1.4.tar.gz
```

建立对应目录

```
 mkdir -p /var/lib/zabbix/percona/scripts/
```

拷贝参数文件到 agentd 的配置文件目录（一般为 zabbix_agent 安装目录下的 etc 下的 zabbix_agentd.conf.d 目录）

```
cd /usr/local/zabbix/etc/zabbix_agentd.conf.d/
cp /opt/percona-monitoring-plugins-1.1.4/zabbix/templates/userparameter_percona_mysql.conf .
```

修改 zabbix_agentd.conf 启用参数文件文件

```
vi /usr/local/zabbix/etc/zabbix_agentd.conf
```

253 行，取消注释

```
Include=/usr/local/zabbix/etc/zabbix_agentd.conf.d/*.conf
```

修改后，重启 agentd

```
service zabbix-agent restart
```

配置 mysql 数据连接,我直接用 root 用户进行连接

```
cd /var/lib/zabbix/percona/scripts/
cp /opt/percona-monitoring-plugins-1.1.4/zabbix/scripts/* .
```

编辑 mysql 连接信息

```
cd /var/lib/zabbix/percona/scripts/
vi ss_get_mysql_stats.php
```

修改为你数据库的实际参数

```
$mysql_user = 'root';
$mysql_pass = '123456';
$mysql_port = 3306;
```

修改好后测试一下

```
/var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh gg
```

如果出现数据说明链接数据没有问题

```
[root@proxy scripts]# /var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh gg
9
[root@proxy scripts]#
```

配置调用,编辑

```
vi /home/zabbix/.my.cnf
```

内容如下

```
[client]
user = root
password = 123456
```

测试一下

```
sudo -u zabbix -H /var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh running-slave
```

如果返回“1”或者“0”，表明成功

```
[root@canghai ~]# sudo -u zabbix -H /var/lib/zabbix/percona/scripts/get_mysql_stats_wrapper.sh running-slave
0
[root@canghai ~]#
```

设置权限

```
chown -R zabbix:zabbix /tmp/localhost-mysql_cacti_stats.txt
```

注意事项：此方法被监控端必须安装 php

# 服务端配置

下载[percona-monitoring-plugins-1.1.4.tar.gz](https://dl.cactifans.com/template/percona-monitoring-plugins-1.1.4.tar.gz)包，并解压，在 zabbix_web 页面选择导入模版，选择 percona-monitoring-plugins-1.1.4 包里 zabbix 目录 templates 目录下的 zabbix_agent_template_percona_mysql_server_ht_2.0.9-sver1.1.4.xml
模版文件
并关联模版到主机
![关联模版](http://www.cactifans.org/wp-content/uploads/2015/04/15.png)
过一会，便可以看到数据已经过来
InnoDB I/O
![InnoDB I/O](http://www.cactifans.org/wp-content/uploads/2015/04/16.png)
Command Counters
![Command Counters](http://www.cactifans.org/wp-content/uploads/2015/04/17.png)
