---
title: Zabbix 5.4自动导出巡检PDF报告
tags: [zabbix, pdf]
categories: [zabbix]
date: 2021-06-17 0:27:30
---

zabbix 5.4版本发布，提供了很多新特性，定时导出PDF报告是一大重点功能。此功能可按照Dashboard维度，定时自动导出报告，并通过邮件发送。
## 安装
zabbix 5.4版本提供了官方的rhel8版本的rpm包，可使用yum方式完成安装。官方未提供的rhel7 rpm包。如需在rhel7 上安装zabbix 5.4需要使用源码编译安装，编译安装注意php版本要求。另外zabbix 5.4版本增加了一个使用go编写的zabbix web service程序，用来实现PDF的生成，此程序编译需要使用go语言编译环境。go语言开发环境配置，请查看[go语言开编译境配置](https://blog.cactifans.com/2020/05/19/Zabbix5.0%E7%89%88%E6%9C%ACAgent2%E5%AE%89%E8%A3%85/#%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85)
### yum
如果系统为rhel8，可使用yum方式安装，注意安装
### 编译
基础的lnmp环境建议使用[lnmp一键安装包配置](https://lnmp.org/download.html) 安装。
下载zabbix 5.4源码，并解压
```
wget https://cdn.zabbix.com/zabbix/sources/stable/5.4/zabbix-5.4.1.tar.gz
tar zxvf zabbix-5.4.1.tar.gz
cd zabbix-5.4.1
```
解压后使用以下命令进行编译
```
./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --enable-webservice --with-mysql=/usr/local/mysql/bin/mysql_config --with-net-snmp --with-libcurl --with-libxml2 
make
make install
```
与之前编译有所不同，5.4版本需要添加--enable-webservice参数，指定添加编译zabbix web service服务。--with-mysql需要指定mysql_config文件位置，一般情况默认即可，这里使用lnmp环境安装不在默认位置，需指定为具体文件位置。其他组件安装与其他版本无异，安装好之后初始化Web页面，并启动zabbix server 确保服务正常。
导出PDF需要使用chrome，按照如下命令安装即可
```
yum install https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm -y
```
### 配置服务
zabbix web service是一个后台服务，编写systemd启动文件
```
vi /lib/systemd/system/zabbix-web-service.service
```
文件内容如下
```
[Unit]
Description=Zabbix Web Service
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_web_service.conf"
EnvironmentFile=-/etc/default/zabbix-web_service
Type=simple
Restart=on-failure
KillMode=control-group
ExecStart=/usr/local/zabbix/sbin/zabbix_web_service -c $CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
User=zabbix
Group=zabbix

[Install]
WantedBy=multi-user.target
```
启动服务
```
systemctl enable --now zabbix-web-service
```
zabbix web service默认配置文件为zabbix_web_service.conf，默认情况下不需要修改。
zabbix server配置文件注意如下配置，并重启zabbix server
```
StartReportWriters=5
WebServiceURL=http://localhost:10053/report
```
## 配置
安装完成后，需要在web页面进行一定的配置，才能生成PDF报告。
配置Frontend URL
![1](https://img.cactifans.com/wp-content/uploads/2021/06/1623741574944.jpg)
这里配置为zabbix web实际访问地址
报告发送需要配置用户邮件媒介，其他媒介会无法发送，使用zabbix 自带的邮件媒介，配置邮件服务器信息。

配置任务，点击Reports菜单下的Scheduled reports，新建调度任务。
![2](https://img.cactifans.com/wp-content/uploads/2021/06/1623743537858.jpg)
配置任务名称，Dashborad、发送时间、选定需要接受的用户或组。
配置完成后点击Test测试。
![3](https://img.cactifans.com/wp-content/uploads/2021/06/1623746386264.jpg)
提示成功，会收到邮件，
![4](https://img.cactifans.com/wp-content/uploads/2021/06/1623744002342.jpg)
附件为生成的PDF报告
![5](https://img.cactifans.com/wp-content/uploads/2021/06/1623746254037.jpg)
### 使用指南
建议按照业务系统或分组为维度，定制不同的Dashboard页面，制定多个巡检报告任务，如天，周，月等，可实现简单的自动化巡检任务。