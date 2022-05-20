---
title: Zabbix4.2 新功能实践 1-时间序列数据库 TimescaleDB
date: 2019-04-04 16:45:08
tags: [zabbix]
categories: [zabbix]
---

4 月 2 号万众期待的 Zabbix4.2 终于发布了！新版本提供了很多特性，接下来几期主要介绍 Zabbix4.2 的一些新特性的使用。本次主要介绍 TimescaleDB。

## TimescaleDB 介绍

TimescaleDB 是基于 PostgreSQL 的时序数据库插件，完全继承了 PostgreSQL 的功能，对于复杂查询，各种类型(GIS,json,k-v,图像特征值,range,数组,复合类型,自定义类型.....)的支持非常丰富，非常适合工业化的时序数据库场景需求。具有以下特点：

1. 基于时序优化
2. 自动分片（按时间、空间自动分片(chunk)）
3. 全 SQL 接口
4. 支持垂直于横向扩展
5. 支持时间维度、空间维度自动分区。空间维度指属性字段（例如传感器 ID，用户 ID 等）
6. 支持多个 SERVER，多个 CHUNK 的并行查询。分区在 TimescaleDB 中被称为 chunk。
7. 自动调整 CHUNK 的大小
8. 内部写优化（批量提交、内存索引、事务支持、数据倒灌）。

内存索引，因为 chunk size 比较适中，所以索引基本上都不会被交换出去，写性能比较好。
数据倒灌，因为有些传感器的数据可能写入延迟，导致需要写以前的 chunk，timescaleDB 允许这样的事情发生(可配置)。

9. 复杂查询优化（根据查询条件自动选择 chunk，最近值获取优化(最小化的扫描,类似递归收敛)，limit 子句 pushdown 到不同的 server,chunks，并行的聚合操作）
10. 利用已有的 PostgreSQL 特性（支持 GIS，JOIN 等），方便的管理（流复制、PITR）
11. 支持自动的按时间保留策略（自动删除过旧数据）

看介绍是很适合监控数据的存储。之前对于监控数据的存储，建议进行分区表操作，进行管理。Zabbix4.2 支持 TimescaleDB 应该说是一个好消息，至于具体性能提升，还有待测试.

## TimescaleDB 安装

TimescaleDB 基于 PostgreSQL，因此我们需要先安装 PostgreSQL，由于之前很少用，因此这里简单描述以下安装过程
基本环境：

| 操作系统        | Centos7.6.1810 x86_64 |
| :-------------- | :-------------------- |
| PostgreSQL 版本 | 11.2                  |
| 安装方式        | yum                   |

由于为测试环境，避免麻烦关闭防火墙，禁用 seLinux，并启动 chrony 时间同步服务。
添加 PostgreSQL 的 yum 源

```bash
yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
```

添加 TimescaleDB 的 yum 源

```bash
cat > /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/7/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL
```

安装包

```bash
yum update -y
yum install -y timescaledb-postgresql-11
```

初始化数据库

```
postgresql-11-setup initdb
```

启动 PostgreSQL

```bash
systemctl start postgresql-11
systemctl enable postgresql-11
```

启动之后，使用以下命令初始化 postgresql 配置文件

```bash
timescaledb-tune
```

初学者建议一切按照推荐数值，全部按 Y 同意即可完成配置。由于我的 Zabbix Server 和 PostgreSQL 为不同机器，因此需要开启 PostgreSQL 远程连接（默认关闭）
修改 PostgreSQL 默认配置文件/var/lib/pgsql/11/data/postgresql.conf
修改 listen 地址为所有地址(即\*)，默认监听 127.0.0.1

```bash
listen_addresses = '*'
```

修改客户端认证配置文件:/var/lib/pgsql/11/data/pg_hba.conf 添加如下

```bash
host    all   all    0.0.0.0/0    md5
```

配置好之后，重启 PostgreSQL 使配置生效

```
systemctl restart postgresql-11
```

至此 TimescaleDB 配置安装完成

## Zabbix 使用 TimescaleDB

目前只有 Zabbix Server 支持 TimescaleDB，Zabbix Proxy 不支持 TimescaleDB。
配置成功 TimescaleDB 之后，建立 Zabbix 相关用户，并导入 Zabbix 数据库

```
sudo -u postgres psql
create user zabbix with password 'zabbixpwd123';
create database zabbix owner zabbix;
grant all privileges on database zabbix to zabbix;
```

以上命令建立名为 zabbix 的用户，密码为 zabbixpwd123，并建立了 zabbix 数据库，并授权于所有权限给 zabbix 用户。
下载 zabbix4.2 源码，并解压,进入 postgresql 目录

```bash
tar zxvf zabbix-4.2.0.tar.gz
cd zabbix-4.2.0/database/postgresql/
```

以此按照顺序，导入数据库

```bash
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
cat schema.sql | sudo -u zabbix psql zabbix
cat images.sql | sudo -u zabbix psql zabbix
cat data.sql | sudo -u zabbix psql zabbix
cat timescaledb.sql | sudo -u zabbix psql zabbix
```

开启成功 TimescaleDb
![1](https://img.cactifans.com/wp-content/uploads/2019/04/70FC7FC8-2EC1-413D-A72F-D8BBD1D4C015-1024x379.jpg)
与平常不同，这里开启了 TimescaleDB 插件支持，并使用 timescaledb.sql 为历史和趋势数据创建了 hypertable 表.hypertable 表是 timescaledb 抽象的 一张表,让用户操作 hypertable 就像 操作 postgres 的普通表一样，在内部,timescaledb 自动将 hypertable 分割成块, timescaledb 会自动操作和管理 hypertable 的分区表，对于用户来说是透明的.create_hypertable 有两个参数,第一个参数是表名，第二个参数 是分区列,一般为 TIMESTAMPTZ 类型.这里看到为历史数据的 clock 列。

```bash
SELECT create_hypertable('history', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('history_uint', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('history_log', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('history_text', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('history_str', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('trends', 'clock', chunk_time_interval => 86400, migrate_data => true);
SELECT create_hypertable('trends_uint', 'clock', chunk_time_interval => 86400, migrate_data => true);
UPDATE config SET db_extension='timescaledb',hk_history_global=1,hk_trends_global=1;
```

完成之后，在 Zabbix Server 里配置相关数据库连接参数即可，与支持 postgresql 的配置一致。

## 基本测试

使用 TimescaleDB 之后，使用我之前一篇 blog[基于 kubernetes 平台的 Zabbix 压力测试](https://blog.cactifans.com/2019/01/22/%E5%9F%BA%E4%BA%8Ekubernetes%E5%B9%B3%E5%8F%B0%E7%9A%84Zabbix%E5%8E%8B%E5%8A%9B%E6%B5%8B%E8%AF%95/) 的方法增加到 5k Nvps
![2](https://img.cactifans.com/wp-content/uploads/2019/04/4B570A96-4559-44FA-BBB3-A9BD7C378EBF.jpg)
并关联 Template App Zabbix Server Stress 1k active 模版，观察机器情况
Zabbix Server
![3](https://img.cactifans.com/wp-content/uploads/2019/04/607BC706-F1BD-402F-98C4-C167A5DA9774-1024x350.jpg)
![4](https://img.cactifans.com/wp-content/uploads/2019/04/1C4D37CC-F84A-47CB-B2FC-C669169C971B-1024x336.jpg)
![5](https://img.cactifans.com/wp-content/uploads/2019/04/1CA9EB8B-5130-493F-B459-316500438801-1024x500.jpg)
TimescaleDB
![6](https://img.cactifans.com/wp-content/uploads/2019/04/04B68CCB-DC92-4867-BEDD-E291EA66D1E7-1024x795.jpg)
![7](https://img.cactifans.com/wp-content/uploads/2019/04/25AE4313-12E7-40EC-A30E-99F96C5A8DA2-1024x514.jpg)

## 总结

本次主要介绍了 Zabbix 使用 TimescaleDB，安装配置比较简单，至于性能是否有大的提升，还需要后续进行测试和验证。
**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
