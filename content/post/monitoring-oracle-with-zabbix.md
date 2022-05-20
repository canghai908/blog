---
title: "monitoring oracle with zabbix"
date: 2015-05-25 21:49:06
tags:
  - zabbix
  - oracle
categories: [zabbix]
---

使用 zabbix 监控 oracle 数据库之前用过[Orabbix](http://www.smartmarmot.com/product/orabbix/)和[Pyora](https://github.com/bicofino/Pyora)，都不太理想，主要问题有以下几点
1.orabbix 需要数据库安装 java 环境，另外 SQL 脚本哪里，本人实在不会配置，调整起来比较麻烦，因为不熟悉
2.Pyora 需要在服务端指定链接的数据库用户名等信息，有点不安全，另外一个 python 的 cx-Oracle 依赖包真心难安装，头疼！
于是打算自己用简单的方法做一个！

## 整体结构

使用 shell 脚本，su 到 oracle 用户，使用 sqlplus 执行 sql 查询语句，把结果保存为 csv 文件，再通过程序处理 csv 文件，添加对应的 key，来使 zabbix 实现监控 oracle 数据库。shell 脚本使用定时任务，每一分钟执行一次，定时导出结果到 CSV 文件

### 优点

1.简单方便，直接使用 sqlplus，无依赖组件 2.自定定义查询的性能 sql 语句，可定制程度高 3.直接在数据库执行，不需要添加用户，授权等操作 4.自动发现表空间，阈值设置。

### 缺点

目前还没有发现^\_^！

## 部署过程

### 被监控端配置

数据库服务器要首先安装 zabbix-agnetd 客户端，部署提取性能的 shell 脚本到数据库机器，并且富于执行权限。
shell 脚本：[aa.sh](https://dl.cactifans.com/tools/aa.sh)(脚本里的 sql 可能有些不专业，大牛无喷啊！^\_^)
下载脚本到 oracle 数据库服务器，目录随意（我下载到 ORACLE_HOME 目录下）

```
mkdir －p /tmp/oracle
chmod -R 777 /tmp/oracle
wget https://dl.cactifans.com/tools/aa.sh
chmod a+x aa.sh
```

切换到 oracle 用户，执行脚本测试以下

```
su - oracle
sh aa.sh
```

执行之后，查看/tmp/oracle 目录下，会生成很多 csv 文件，如果没有生成，查看 sqlplus 是否能执行，或者目录是否有写入权限

```
ls -lh /tmp/oracle
```

可看到如下，表示正确执行
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/1-1024x553.png)
可 cat 查看文件是否有结果，如下
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/2-1024x252.png)
确认无问题之后，添加到计划任务

```
 vi /etc/crontab
 */1  *  *  *  * root su - oracle -c /home/oracle/aa.sh
```

表示一分钟执行一次脚本，导出结果到 csv 文件
下载 csv 处理文件到数据库服务器，处理 csv 文件可用 shell，python 等语言写，由于本人对以上二种语言都不熟悉，所以用自己还算熟悉的 go 写了一个，就是文件有点大！
32 位系统 csv 处理文件：[zabbix_oracle.x86.tar.gz](https://dl.cactifans.com/tools/zabbix_oracle.x86.tar.gz)
64 位系统 csv 处理文件：[zabbix_oracle.x86_64.tar.gz](https://dl.cactifans.com/tools/zabbix_oracle.x86_64.tar.gz)
安装文件到 zabbix_agentd 的 bin 目录下（可随意选择，赋予可执行权限即可）
更新记录：
1.2015.11.20 更新处理文件 bug 2.更新下载地址
64 位操作系统下演示

```
wget https://dl.cactifans.com/tools/zabbix_oracle.x86_64.tar.gz
tar zxvf abbix_oracle.x86_64.tar.gz
mv oracle /usr/local/zabbix/bin
chown zabbix:zabbix /usr/local/zabbix/bin/oracle
chmod a+x /usr/local/zabbix/bin/oracle/*
```

修改 zabbix_agentd.conf 文件，添加对应的 key

```
vi /usr/local/zabbix/etc/zabbix_agentd.conf
UnsafeUserParameters=1
UserParameter=oracle.tablespace.discovery,/usr/local/zabbix/bin/oracle/tablespace_name
UserParameter=oracle.tablespace[*],/usr/local/zabbix/bin/oracle/tablespace_space $1 $2
UserParameter=oracle.one[*],/usr/local/zabbix/bin/oracle/oracle_one $1
UserParameter=oracle.process,/usr/local/zabbix/bin/oracle/process
UserParameter=oracle.tablespaceinc[*],/usr/local/zabbix/bin/oracle/tablespace_inc $1
UserParameter=oracle.sga[*],/usr/local/zabbix/bin/oracle/sga $1
UserParameter=oracle.version,/usr/local/zabbix/bin/oracle/version
UserParameter=oracle.connect,/usr/local/zabbix/bin/oracle/connect
```

添加好之后如下
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/3-1024x187.png)
添加好之后，重启 zabbxi－agent，到服务端测试一下

```
/usr/local/bin/zabbix_get -s 192.168.7.186 -k oracle.version
```

![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/4-1024x80.png)
如果能输出，表示工作正常（192.168.7.186 为我的 oracle 服务器地址）

### zabbix 服务端配置

本来要在 zabbix 上添加 key 的，有点麻烦，索性坐成了一个模版，大家可直接下载导入即可，zabbix2.4.4 版本上导出，不知是否能够在通用！
模版文件[：zabbix_oracle_templates.tar.gz](https://dl.cactifans.com/oracle/zabbix_oracle_templates.tar.gz)
下载后直接导入 zabbix，关联模版数据库服务器，即可。第一次做模版，有不到之处望大家指出。
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/5-1024x414.png)

## 最终效果

表空间自动发现
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/6-1024x625.png)
SGA
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/7-1024x175.png)
Session
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/8-1024x147.png)
Dbinfo
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/9-1024x195.png)
Tablespace
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/10-1024x483.png)
Tablespace_free_pct
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/11-1024x476.png)
Tablespace_free_pct_trigger
![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/05/11-1024x476.png)
