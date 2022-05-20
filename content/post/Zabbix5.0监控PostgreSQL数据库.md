---
title: "Zabbix 5.0 监控 PostgreSQL 数据库"
date: 2020-07-18T17:21:00+08:00
tags: [postgresql]
categories: [zabbix]
---

Zabbix 支持 PostgreSQL 作为后台数据库，相比 Mysql，PostgreSQL 可加载 timescaledb 插件，提升 Zabbix 性能，同时还支持数据的压缩，因此对于 PostgreSQL 数据库的监控是非常需要的。使用 zabbix5.0 自带的数据库模版及脚本即可实现对 PostgreSQL 的监控

## 一.PostgreSQL 配置

## 1.创建用户

需要在 PostgreSQL 数据库建立监控专用的用户，由于 PostgreSQL 版本不同相关命令会有一定差别，创建一个 zbx_monitor 用户密码为 zbx_monitorpwd123
PostgreSQL 10 以上版本

```
su - postgres
psql
CREATE USER zbx_monitor WITH PASSWORD 'zbx_monitorpwd123' INHERIT;
GRANT pg_monitor TO zbx_monitor;
```

PostgreSQL 9.6 版本及以下

```
su - postgres
psql
CREATE USER zbx_monitor WITH PASSWORD 'zbx_monitorpwd123';
GRANT SELECT ON pg_stat_database TO zbx_monitor;
ALTER USER zbx_monitor WITH SUPERUSER;
```

## 2.配置访问策略

编辑 pg_hba.conf 文件，并添加如下内容

```
host all zbx_monitor 127.0.0.1/32 trust
host all zbx_monitor 0.0.0.0/0 md5
host all zbx_monitor ::0/0 md5
```

如果 Zabbix agent 和 PostgreSQL 在不同机器，需要配置密码文件，需要创建.pgpass 文件，并存放在 zabbix 用户的家目录下，内容如下：

```
<REMOTE_HOST1>:5432:postgres:zbx_monitor:<PASSWORD>
```

配置好之后记得重启 PostgreSQL 服务

## 二.Zabbix 配置

## 1.添加主机

添加主机之后，关联系统自带的 Template DB PostgreSQL 模版
![1](https://img.cactifans.com/wp-content/uploads/2020/07/75A5F81D-B1D2-450C-A0A2-2D7585395F89.png)
配置如以下主机宏
![1](https://img.cactifans.com/wp-content/uploads/2020/07/992B3634-C89B-4404-98F0-6449A8AAEF8A.png)

| 宏名                            | 数值        | 描述                           |
| :------------------------------ | :---------- | :----------------------------- |
| {\$PG.HOST}                     | 127.0.0.1   | 主机 IP                        |
| {\$PG.PORT}                     | 5432        | 端口                           |
| {\$PG.USER}                     | zbx_monitor | 监控用户                       |
| {\$PG.DB}                       | postgres    | 连接默认数据库                 |
| {\$PG.LLD.FILTER.DBNAME}        | (.\*)       | 自动发现数据库过滤，默认不过滤 |
| {\$PG.CHECKPOINTS_REQ.MAX.WARN} | 5           | checkpoints 阈值               |

更多宏的配置及说明请查看官网说明[https://www.zabbix.com/integrations/postgresql](https://www.zabbix.com/integrations/postgresql)

## 2.配置 Agent

PostgreSQL 监控需要在 Zabbix Agent 端添加脚本文件。
从https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/db/postgresql 下载监控脚本，官方下载较慢，我已转存到这个地址，下载地址
[https://dl.cactifans.com/zabbix/postgresql.tar.gz](https://dl.cactifans.com/zabbix/postgresql.tar.gz)
下载到 Agent 所在机器，添加 Postgresql 监控 SQL 文件

```bash
wget https://dl.cactifans.com/zabbix/postgresql.tar.gz
mkdir -p /var/lib/zabbix/
tar zxvf postgresql.tar.gz
cp -r postgresql/postgresql/ /var/lib/zabbix/
```

添加 UserParameter 文件到 Agent 的 zabbix_agentd.d 目录（根据实际情况修改）

```
cp -r postgresql/template_db_postgresql.conf  /etc/zabbix/zabbix_agentd.d/
```

并修改 zabbix_agentd.conf 文件，确保 UserParameter 被加载

```
Include=/etc/zabbix/zabbix_agentd.conf.d/*.conf
```

重启 Zabbix Agent

## 3.脚本 Bug 修复

正常添加后，已经可以看到部分数据，观察 Zabbix Server 日志，会发现如下错误信息

```
29128:20200702:014602.152 item "Zabbix server:pgsql.dbstat.blks_read.rate["zabbix"]" became not supported: Preprocessing failed for: psql:/var/lib/zabbix/postgresql/pgsql.dbstat.sql:17: ERROR:  field name must not be null
1. Failed: cannot extract value from json by path "$['zabbix'].blks_read": cannot parse as a valid JSON object: invalid object format, expected opening character '{' or '[' at: 'psql:/var/lib/zabbix/postgresql/pgsql.dbstat.sql:17: ERROR:  field name must not be null'
```

观察 PostgreSQL 监控指标，确实有部分 item 显示为 unsupported 状态，查询资料后发现是由于 sql 脚本问题，修改 sql 文件

```
vi /var/lib/zabbix/postgresql/pgsql.dbstat.sql
```

修改为如下内容

```sql
SELECT json_object_agg(datname, row_to_json(T)) FROM (
        SELECT datname,
                        numbackends,
                        xact_commit,
                        xact_rollback,
                        blks_read,
                        blks_hit,
                        tup_returned,
                        tup_fetched,
                        tup_inserted,
                        tup_updated,
                        tup_deleted,
                        conflicts,
                        temp_files,
                        temp_bytes,
                        deadlocks
        FROM pg_stat_database
        WHERE datname is not NULL
) T
```

等待一会就会发现如下日志

```
 29128:20200702:104902.227 item "Zabbix server:pgsql.dbstat.tup_deleted.rate["zabbix"]" became supported
 29128:20200702:104902.227 item "Zabbix server:pgsql.dbstat.tup_fetched.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.tup_inserted.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.tup_returned.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.tup_updated.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.xact_commit.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.xact_rollback.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.temp_bytes.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.blks_read.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.conflicts.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.blks_hit.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.temp_files.rate["zabbix"]" became supported
 29128:20200702:104902.228 item "Zabbix server:pgsql.dbstat.deadlocks.rate["zabbix"]" became supported
```

采集正常，此 bug 已提交官方[ZBX-18099](https://support.zabbix.com/browse/ZBX-18099)

## 三.最终效果

基本信息
![1](https://img.cactifans.com/wp-content/uploads/2020/07/C5F8D5A871DEB1A8CBEA153A15AF1327.jpg)
自动发现数据库，监控数据库信息
![1](https://img.cactifans.com/wp-content/uploads/2020/07/4A1D5B87-27A0-4F3D-9134-6A4F4F754B5D.png)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
