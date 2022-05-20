---
title: Zabbix6.0版本TimescaleDB功能体验
date: 2022-01-15 00:58:06
tags: [6.0, TimescaleDB]
categories: [zabbix]
---

## 前言
Zabbix 6.0目前已发布beta1版本，包含众多新功能和新特性，本文主要介绍6.0 配置TimescaleDB，此安装配置方法可基本通用与其他版本。
## TimescaleDB

TimescaleDB基于PostgreSQL数据库打造的一款时序数据库，插件化的形式部署，随着PostgreSQL的版本升级而升级，具备以下特点：
1. 基于时序优化；
2. 自动分片（按时间、空间自动分片(chunk)）；
3. 全SQL接口；
4. 支持垂直与横向扩展；
5. 支持时间维度、空间维度自动分区。空间维度指属性字段（例如传感器ID，用户ID等）；
6. 支持多个SERVER，多个CHUNK的并行查询。分区在TimescaleDB中被称为chunk；
7. 自动调整CHUNK的大小；
8. 内部写优化（批量提交、内存索引、事务支持、数据倒灌）；
9. 复杂查询优化（根据查询条件自动选择chunk，最近值获取优化(最小化的扫描,类似递归收敛)，limit子句pushdown到不同的；server,chunks，并行的聚合操作）；
10. 利用已有的PostgreSQL特性（支持GIS，JOIN等），方便的管理（流复制、PITR）；
11. 支持自动的按时间保留策略（自动删除过旧数据）；

Zabbix 从5.0版本开始全面支持TimescaleDB，并针对其特性做了优化。可自动压缩历史数据存储，节省50-70%的存储空间，同时具备自动分区功能。通过Zabbix Housekeeper清理历史数据时直接清理对应的分区，大大提高了历史数据的清理效率。建议新建系统采用TimescaleDB方案。

## 环境介绍
| 角色      |     配置 |   主机名  |IP      |版本 |  
| :-------- | :-------- | :------   |:------ |  :------ |  
| Zabbix Server   |   4v8G40G  CentOS 8.5_x64|  zbx-srv-61  |172.16.66.61 |Zabbix 6.0 beta1 |  
| Zabbix Web  	 |    4v8G40G  CentOS 8.5_x64 | zbx-web-63  |172.16.66.63 |Apache 2.4.3/PHP 7.2.24 |  
| Zabbix DB		|    4v8G40G  CentOS 8.5_x64|  zbx-db-64  | 172.16.66.64|PostgreSQL 13/TimescaleDB 2 |  

此为实验环境，生产环境建议按照实际需要调整机器硬件配置。所有机器配置时间同步，并添加对应的hosts

```
172.16.66.61 zbx-srv-61
172.16.66.63 zbx-web-63
172.16.66.64 zbx-db-61
```
## Zabbix DB
安装好系统对系统进行初始化配置，安装必要的包.
## 初始化

```bash
yum update -y
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
dnf install chrony wget -y
systemctl enable --now chronyd
setenforce 0
```
zabbix 6.0目前支持postgresql  13不支持最新的14版本，本次使用postgresql 13+Timescaledb。
## 安装postgresql 
```
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql13-server
```
## 安装Timescaledb
添加Timescaledb源
```
tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/$(rpm -E %{rhel})/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL
```
安装TimescaleDB包
```
dnf install timescaledb-2-postgresql-13 -y
```
## 配置
初始化pgsql
```
/usr/pgsql-13/bin/postgresql-13-setup initdb
```
启动pgsql server
```
systemctl enable --now postgresql-13
```
添加Timescaledb并配置参数
```
timescaledb-tune --pg-config=/usr/pgsql-13/bin/pg_config
```
会出现交互画面，一路y 即可，此步骤会根据当前机器配置，调整pgsql配置参数，并加载Timescaledb插件库.
重启pgsql生效.
```
systemctl restart postgresql-13
```
建立zabbix用户及数据库
```
sudo -u postgres createuser --pwprompt zabbix
```
此处是需要输入数据库zabbix用户的密码，输入二次后确认。此处配置密码为: zabbixpwd_123
后续zabbix server连接数据库使用这个密码，用户为zabbix
创建zabbix数据库
```
sudo -u postgres createdb -O zabbix zabbix
```
为zabbix数据库启用timescledb插件
```
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
systemctl restart postgresql-13
```
成功后会出现如下画面，表示配置完成。
![1](https://img.cactifans.com/wp-content/uploads/2022/01/20220105180905.png)
下载zabbix 6.0beta1源码并导入数据库。
```
wget https://cdn.zabbix.com/zabbix/sources/development/6.0/zabbix-6.0.0beta1.tar.gz
tar zxvf zabbix-6.0.0beta1.tar.gz
cd zabbix-6.0.0beta1/database/postgresql/
useradd zabbix
```
依次按照顺序导入三个zabbix sql文件
```
cat schema.sql |sudo -u zabbix psql zabbix
cat images.sql |sudo -u zabbix psql zabbix
cat data.sql |sudo -u zabbix psql zabbix
```
导入timescledb表配置sql
```
cat timescaledb.sql |sudo -u zabbix psql zabbix
```
导入成功后会后如下提示
![2](https://img.cactifans.com/wp-content/uploads/2022/01/20220105180911.png)
修改配置允许远程连接
```
sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/g" /var/lib/pgsql/13/data/postgresql.conf
sed -i 's/#port = 5432/port = 5432/g' /var/lib/pgsql/13/data/postgresql.conf
sed -i 's/max_connections = 100/max_connections = 500/g' /var/lib/pgsql/13/data/postgresql.conf
```
连接数修改成500，生产可根据实际情况修改。
配置使用md5方式认证
```
vi /var/lib/pgsql/13/data/pg_hba.conf
```
添加如下信息到# IPv4 local connections之后
```
host    all             all             0.0.0.0/0               md5
```
重启pgsql
```
systemctl restart postgresql-13
```
## Zabbix Server
Zabbix Server使用源码编译方式安装，其他版本可参考此安装方式。
## 初始化
```bash
yum update -y
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
dnf --enablerepo=powertools install OpenIPMI-devel -y
dnf install make wget chrony gcc curl-devel net-snmp-devel \
libxml2-devel libevent-devel pcre-devel -y
setenforce 0
systemctl enable --now chronyd
```
由于后端采用pgsql数据库，因此需要安装pgsql的开发包。
```
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql13-devel -y
```
## 编译
下载zabbix server源码
```
useradd zabbix
wget https://cdn.zabbix.com/zabbix/sources/development/6.0/zabbix-6.0.0beta1.tar.gz
tar zxvf zabbix-6.0.0beta1.tar.gz
cd zabbix-6.0.0beta1/
```
编译
```
./configure --prefix=/usr/local/zabbix --enable-server --enable-agent \
--with-postgresql=/usr/pgsql-13/bin/pg_config  --with-net-snmp \
--with-libcurl --with-libxml2 --with-openipmi
```
此处会检测各种依赖组件及配置，如果出现error根据错误安装对应组件，无误后最终会输出所要安装的内容。
![3](https://img.cactifans.com/wp-content/uploads/2022/01/20220105180807.png)
执行make
```
make
make install
```
## 配置
安装后，需要修改配置文件里的数据库连接信息
```
sed -i 's/# DBHost=localhost/DBHost=172.16.66.64/g' /usr/local/zabbix/etc/zabbix_server.conf
sed -i 's/# DBPassword=/DBPassword=zabbixpwd_123/g' /usr/local/zabbix/etc/zabbix_server.conf
```
Zabbix 6.0增加了二个关于HA的配置参数，建议配置
```
sed -i 's/# HANodeName=/HANodeName=zbx-srv-61/g' /usr/local/zabbix/etc/zabbix_server.conf
sed -i 's/# NodeAddress=localhost:10051/NodeAddress=172.16.66.61:10051/g' /usr/local/zabbix/etc/zabbix_server.conf
```
HANodeName修改为主机名，这里最好配置唯一；
NodeAddress为节点地址，这里配置为实际ip+默认的10050端口；
创建zabbix server启动脚本
```
tee /lib/systemd/system/zabbix-server.service <<EOL
[Unit]
Description=Zabbix Server
After=syslog.target
After=network.target
After=mysql.service
After=mysqld.service
After=mariadb.service
After=postgresql.service

[Service]
Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_server.conf"
EnvironmentFile=-/etc/sysconfig/zabbix-server
Type=forking
Restart=on-failure
PIDFile=/tmp/zabbix_server.pid
KillMode=control-group
ExecStart=/usr/local/zabbix/sbin/zabbix_server -c \$CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
TimeoutSec=0

[Install]
WantedBy=multi-user.target
EOL
```
创建zabbix agent启动脚本
```
tee /lib/systemd/system/zabbix-agent.service <<EOL
[Unit]
Description=Zabbix Agent
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_agentd.conf"
EnvironmentFile=-/etc/sysconfig/zabbix-agent
Type=forking
Restart=on-failure
PIDFile=/tmp/zabbix_agentd.pid
KillMode=control-group
ExecStart=/usr/local/zabbix/sbin/zabbix_agentd -c \$CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
User=zabbix
Group=zabbix

[Install]
WantedBy=multi-user.target
EOL
```
## 启动
使用以下命令启动zabbix server及zabbix agent
```
systemctl enable --now zabbix-server
systemctl enable --now zabbix-agent
```
如启动异常，查看日志确认异常，日志位置/tmp/zabbix_server.log

## Zabbix WEB
Zabbix 6.0需要php最低版本为7.2，由于使用pgsql，因此需要按照php的pgsql扩展组件。
## 初始化
```bash
yum update -y
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
dnf install wget chrony httpd php php-pgsql php-xml php-ldap php-json php-gd php-mbstring php-bcmath langpacks-zh_CN.noarch -y
systemctl enable --now chronyd
setenforce 0
```
## 安装web
下载zabbix源码，并拷贝到对应目录
```
wget https://cdn.zabbix.com/zabbix/sources/development/6.0/zabbix-6.0.0beta1.tar.gz
tar zxvf zabbix-6.0.0beta1.tar.gz
cd zabbix-6.0.0beta1/ui
cp -raf * /var/www/html/
```
修改php参数，并启动apache和php
```
sed -i 's/post_max_size = 8M/post_max_size = 16M/' /etc/php.ini
sed -i 's/max_execution_time = 30/max_execution_time = 300/' /etc/php.ini
sed -i 's/max_input_time = 60/max_input_time = 300/' /etc/php.ini
chown -R apache:apache /var/www/html/
systemctl enable --now httpd
systemctl enable --now php-fpm
```
## Web初始化
浏览器输入Web IP地址：http://172.16.66.63/  这里有个提示，不用关注
![7](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_103743.png)
确保这里全部ok再下一步，
![8](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104250.png)
这里填入数据库服务器地址及配置的数据库密码zabbixpwd_123，并取消使用TLS连接
![9](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104329.png)
这里zabbix server name留空即可，选择对应的时区，这里选择Asia/Shanghai
![10](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-11_123132.png)
确认无误后点击Next
![11](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104438.png)
创建文件成功，如失败可能是web目录没有写入权限
![12](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104444.png)
使用默认的帐号密码登陆，帐号：Admin 密码：zabbix
![13](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104457.png)
首页
![14](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-05_104522.png)
安装成功
## 基本设置
安装完之后，需要对系统进行一些配置Administrator-Greneral-Housekeeping
这里可配置history（详情）数据与Trend（趋势）数据保留的时间。默认已开启7天历史数据压缩。
![14](https://img.cactifans.com/wp-content/uploads/2022/01/2022-01-11_125749.png)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
