---
title: Zabbix 5.0使用TimescaleDB数据库
tags: [zabbix, timescaledb]
categories: [timescaledb]
date: 2020-10-12 0:27:30
---

## Zabbix 5.0使用TimescaleDB数据库
zabbix5.0版本支持TimescaleDB作为后端数据库，可提供数据自动分区、自动数据清理、数据压缩等特性，以下介绍通过yum方式安装zabbix5.0版本并使用TimescaleDB作为后端数据库

### 环境介绍

| 环境      |    版本 | 
| :-------- |:--------| 
| 操作系统  | Centos 7.8 x86_64|  
| 数据库  | PostgreSQL 12.4|  
|TimescaleDB  | TimescaleDB 1.7.4|  
| Zabbix  | Zabbix server 5.0.4|  

### 初始化系统
更新系统
```
yum update -y
```
关闭selinux/firewall
```
systemctl disable --now firewalld
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
reboot
```
安装基础组件
```
yum install lrzsz wget unzip screen chrony yum-utils -y
```
开启时间同步
```
systemctl enable --now chronyd
```

### 安装Zabbix Server
安装zabbix源
```
rpm -Uvh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
```
默认源为官方源地址访问较慢切换为阿里云源
```
sed -i "s#http://repo.zabbix.com#https://mirrors.aliyun.com/zabbix#g"  /etc/yum.repos.d/zabbix.repo
```
安装zabbix server及agent
```
yum install zabbix-server-pgsql zabbix-agent –y
```
### 安装zabbix web
zabbix5.0需要高版本php，因此需要安装scl源，安装scl源
```
yum install centos-release-scl –y
```
默认zabbix源禁用了前端源，启用
```
vi /etc/yum.repos.d/zabbix.repo
```
找到[zabbix-frontend]段，enabled修改为1
```
enabled=1
```
安装zabbix web
```
yum install zabbix-web-pgsql-scl zabbix-nginx-conf-scl -y
```
修改nginx配置文件
```
vi /etc/opt/rh/rh-nginx116/nginx/conf.d/zabbix.conf
```
主要修改端口及server_name字段
```
listen 8099;
server_name zabbix2020.com;
```
修改默认端口为8099，可根据实际需要修改
修改php参数
```
vi /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
```
主要修改时区和ACL
```
listen.acl_users = apache,nginx
php_value[date.timezone] = Asia/Shanghai
```
### 安装PostgreSQL

使用yum方式安装PostgreSQL，首先安装PostgreSQL源

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

安装PostgreSQL
```
yum install -y postgresql12-server
```
安装之后需要初始化数据库
```
/usr/pgsql-12/bin/postgresql-12-setup initdb
```
启动PostgreSQL 
```
systemctl enable --now postgresql-12 
```
### 安装TimescaleDB插件

使用yum方式安装timesacledb，首先安装timesacledb源
```
tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
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
更新
```
yum update –y
```
安装timescaledb
```
yum install -y timescaledb-postgresql-12
```
安装之后使用以下脚本进行参数初始化，一切按照默认，按y完成初始化
```
timescaledb-tune --pg-config=/usr/pgsql-12/bin/pg_config
```
重启pg数据库
```
systemctl restart postgresql-12 
```
### Zabbix数据库准备
建立zabbix用户

```
sudo -u postgres createuser --pwprompt zabbix
```

配置zabbix用户的密码为zabbixpwd123
建立zabbix数据库
```
sudo -u postgres createdb -O zabbix zabbix
```

开启timescaledb 插件
```
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
```
开启后会出现提示
导入zabbix基础SQL
```
zcat /usr/share/doc/zabbix-server-pgsql*/create.sql.gz | sudo -u zabbix psql zabbix
```
导入timescaledb配置SQL
```
zcat /usr/share/doc/zabbix-server-pgsql*/timescaledb.sql.gz | sudo -u zabbix psql zabbix
```
### PostgreSQL配置
需要对PostgreSQL进行一些基本的参数配置,默认PostgreSQL不支持远程连接，需要配置
修改配置文件
```
vi /var/lib/pgsql/12/data/postgresql.conf
```
主要配置监听地址及端口等信息
```
listen_addresses = '*’
port = 5432                           
max_connections = 500
```
配置使用md5方式认证
```
vi /var/lib/pgsql/12/data/pg_hba.conf
```
添加如下信息到# IPv4 local connections之后
```
host    all             all             0.0.0.0/0               md5
```
重启PostgreSQL
```
systemctl restart postgresql-12
```
### Zabbix配置
修改zabbix server配置文件中的数据库信息
```
vi /etc/zabbix/zabbix_server.conf
```
主要修改如下，其他默认即可
```
DBHost=127.0.0.1
DBPassword=zabbixpwd123
```

### 重启所有服务并配置开机自启动
```
systemctl restart zabbix-server zabbix-agent rh-nginx116-nginx rh-php72-php-fpm
systemctl enable zabbix-server zabbix-agent rh-nginx116-nginx rh-php72-php-fpm
```
### Web初始化
使用http://ip:8099访问zabbix，点击下一步
![11](https://img.cactifans.com/wp-content/uploads/2020/10/11.png)
确保所有检查都为ok，点击下一步
![22](https://img.cactifans.com/wp-content/uploads/2020/10/22.png)
填入配置的zabbix数据库用户密码，schema为空
![33](https://img.cactifans.com/wp-content/uploads/2020/10/33.png)
默认直接点击下一步
![44](https://img.cactifans.com/wp-content/uploads/2020/10/44.png)
确认信息无误后直接点击下一步
![55](https://img.cactifans.com/wp-content/uploads/2020/10/55.png)
初始化完成点击完成
![66](https://img.cactifans.com/wp-content/uploads/2020/10/66.png)
默认用户：Admin 默认密码：zabbix
![77](https://img.cactifans.com/wp-content/uploads/2020/10/77.png)
zabbix 5.0首页
![88](https://img.cactifans.com/wp-content/uploads/2020/10/88.png)
至此zabbix 5.0安装完成，可在Administration-General-Housekeeping选项中开启数据压缩
![99](https://img.cactifans.com/wp-content/uploads/2020/10/99.png)