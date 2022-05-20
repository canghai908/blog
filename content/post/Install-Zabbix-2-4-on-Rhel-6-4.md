---
title: "Install Zabbix 2.4 on Rhel 6.4"
date: 2015-04-01 14:07:34
tags: [zabbix, rhel]
categories: [zabbix]
---

# 系统安装

通过 iso 安装 rhel6.4 系统 64 位，一路默认即可，设置好网络。

## 注意事项：

1.不使用 UTC 时区 2.选择安装基本服务器 3.安装好之后同步时间：

通过 ntp 同步服务器时间，这里我使用上海交通大学的 ntp 服务器，进行时间同步，执行一下命令同步时间

```bash
ntpdate ntp.sjtu.edu.cn
```

### 4.关闭 seLinux 和防火墙

```bash
vi /etc/sysconfig/selinux
```

把 enforcing 改成 disabled，保存

```
SELINUX=disabled
```

关闭防火墙

```bash
chkconfig iptables off
chkconfig ip6tables off
/etc/init.d/iptables stop
/etc/init.d/ip6tables stop
```

之后重启服务器

## 配置本地 yum 源

由于 rhel 不能直接使用 yum 来完成依赖包自动安装，所以需要配置本地光盘 iso，需要安装 iso 镜像文件，我是将 iso 文件下载到系统里，在系统里挂载，进行 yum 安装，你也可以直接挂在到光驱。
挂在 iso 文件到系统的 mnt 目录下

```bash
mount -t iso9660 -o loop /opt/rhel-server-6.4-x86_64-dvd.iso /mnt/
```

挂载之后编辑 repo 文件

```bash
vi /etc/yum.repos.d/CentOS-Media.repo
```

文件内容如下

```
[c6-media]
name=CentOS-$releasever - Media
baseurl=file:///mnt/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

配置好之后,即可通过 yum 安装 rpm 包，并会自动解决包依赖问题

# 安装 zabbix

## 下载 zabbix 源码包

下载 zabbix 源码包到系统/opt 目录下并解压

```bash
cd /opt
wget http://211.161.151.133/files/122700000397C90D/tianjin.mycodes.net/201502/zabbix-2.4.4.tar.gz
tar zxvf zabbix-2.4.4.tar.gz
```

## 添加 zabbix 用户

```bash
groupadd zabbix
useradd -g zabbix zabbix
```

## 安装依赖包

```
yum install httpd php mysql mysql-server mysql-devel php-gd gcc php-mysql php-xml libcurl-devel curl-* net-snmp* libxml2-* -y
```

还需要安装另外 2 个包，yum 源里是没有的，需要用 rpm 命令安装
[php-bcmath-5.3.3-22.el6.x86_64.rpm ](https://dl.cactifans.com/tools/php-bcmath-5.3.3-22.el6.x86_64.rpm)
[php-mbstring-5.3.3-22.el6.x86_64.rpm ](https://dl.cactifans.com/tools/php-mbstring-5.3.3-22.el6.x86_64.rpm)
下载并安装

```bash
cd /opt
wget https://dl.cactifans.com/tools/tools/php-bcmath-5.3.3-22.el6.x86_64.rpm
wget https://dl.cactifans.com/tools/php-mbstring-5.3.3-22.el6.x86_64.rpm
rpm -ivh php-bcmath-5.3.3-22.el6.x86_64.rpm
rpm -ivh php-mbstring-5.3.3-22.el6.x86_64.rpm
```

## 编译配置

查看编译参数

```bash
cd /opt/zabbix-2.4.4
./configure --help
```

增加--help 参数可查看具体的编译参数，这里我使用官方提供的编译参数进行执行编译

```bash
./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
```

参数说明

> --prefix=/usr/local/zabbix 为指定安装目录为/usr/local/zabbix
> --enable-server 为安装 zabbix 服务端程序
> --enable-agent 为安装 agent 程序
> --with-mysql 为使用 mysql 数据库
> --enable-ipv6 为启用 ipv6 支持
> --with-net-snmp 为启用 snmp 支持
> --with-libcurl 为启用 curl
> --with-libxml2 编译 xml 模块，主要用于监控 vm 虚拟机

编译并执行安装

```
make install
```

## 配置 mysql 数据库

启动 apache 服务和 mysql 数据库，并设置开机自启动

```bash
/etc/init.d/httpd start
/etc/init.d/mysqld start
chkconfig httpd on
chkconfig mysqld on
```

设置 mysql 数据库编码

```
vi /etc/my.cnf
```

在[mysqld]段里添加如下内容

```
 character-set-server=utf8
```

在文件末尾添加如下内容

```
[mysql]
default-character-set = utf8
```

最终的 my.cnf 文件内容

```bash
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
character-set-server=utf8
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
[mysql]
default-character-set = utf8
```

修改之后保存，重新启动 mysql 数据库

```
/etc/init.d/mysqld restart
```

对 mysql 数据库进行安全设置，直接运行一下命令

```
mysql_secure_installation
```

执行后

```
[root@canghai opt]#mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MySQL
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!


In order to log into MySQL to secure it, we'll need the current
password for the root user.  If you've just installed MySQL, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MySQL
root user without the proper authorisation.

Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MySQL installation has an anonymous user, allowing anyone
to log into MySQL without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] y
 ... Success!

By default, MySQL comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...



All done!  If you've completed all of the above steps, your MySQL
installation should now be secure.

Thanks for using MySQL!
```

建立数据库及用户

```
mysql -uroot -p
create database zabbix;
grant all on zabbix.* to zabbix@localhost identified by 'zabbixpwd123';
```

说明

> 建立数据库名为 zabbix
> 创建里 zabbix 用户，并把 zabbix 数据库的所有权限授权给 zabbix 用户
> 设置 zabbix 用户密码为 zabbixpwd123
> 只运行从 localhost（本机）连接 mysql 数据库

导入 zabbix 数据库

```mysql
mysql -uzabbix -p zabbix </opt/zabbix-2.4.4/database/mysql/schema.sql
mysql -uzabbix -p zabbix </opt/zabbix-2.4.4/database/mysql/images.sql
mysql -uzabbix -p zabbix </opt/zabbix-2.4.4/database/mysql/data.sql
```

说明

> 导入必须按照先后顺序导入，先导入数据库结构 schema.sql，再导入 images.sql，最后导入 data.sql，不然会报错
> 密码为刚才设置的 zabbix 用户密码

## 配置 zabbix_server.conf 文件

修改配置文件,主要修改以下几个地方

```
vi /usr/local/zabbix/etc/zabbix_server.conf
```

52 行：开启 debug 模式，方便错误调试
去掉注释修改为 0

```
 DebugLevel=0
```

68 行：数据库连接主机

```
#DBHost=localhost
```

去掉注释

```
DBHost=localhost
```

94 行：连接用户名

```
DBUser=root

```

修改为我们刚才建立的 zabbix 用户

```
DBUser=zabbix
```

102 行：连接密码

```
 # DBPassword=
```

去掉注释，修改为刚才所设置的密码

```
 DBPassword=zabbixpwd123
```

117 行：数据库端口

```
# DBPort=3306
```

去掉注释（不修改也可）

```
 DBPort=3306
```

## 配置前端 web 页面

前端页面在源码的 frontends/php 目录下

```
cp -R /opt/zabbix-2.4.4/frontends/php/* /var/www/html/
```

设置权限

```
chown -R apache:apache /var/www/html
```

打开浏览器，访问http://10.211.55.5（ip为你服务器ip）
![安装页面](http://www.cactifans.org/wp-content/uploads/2015/04/1.png)
就看到了安装的页面，点击 Next
![环境检查](http://www.cactifans.org/wp-content/uploads/2015/04/2.png)
检查页面里面是否有红色的标记，表示环境不符合要求，需要调整 php.ini 文件

```
vi /etc/php.ini
```

文件 729 行，post_max_size 从默认的 8MB，修改为 16MB

```
post_max_size = 16M
```

文件 440 行， max_execution_time = 30 从默认的 30，修改为 300

```
max_execution_time = 300
```

文件 449 行， max_input_time = 60 从默认的 60，修为为 300

```
max_input_time = 300
```

文件 946 行，date.timezone 去掉注释，并设置为 Asia/Shanghai

```
date.timezone = Asia/Shanghai
```

设置好之后，保存，并重启 apache 服务器，使设置生效

```
/etc/init.d/httpd restart
```

返回，安装界面，点击“Retry”，重新检查
![检查](http://www.cactifans.org/wp-content/uploads/2015/04/3.png)
如果环境满足安装要求，就会出现“OK”，点击“Next”，接着安装
![设置数据库](http://www.cactifans.org/wp-content/uploads/2015/04/4.png)
填写 mysql 数据库信息，点击“Test connection”,测试数据库连接，如果没有问题，会显示“OK”，点击“Next”，进入下一步
![设置主机名](http://www.cactifans.org/wp-content/uploads/2015/04/5.png)
设置主机名，端口等，直接默认设置就可以，点击“Next”，
![确认数据库信息](http://www.cactifans.org/wp-content/uploads/2015/04/6.png)
确认数据信息，直接点“Next”
![完成安装](http://www.cactifans.org/wp-content/uploads/2015/04/7.png)
穿件配置文件成功，表明安装成功，点“Finish”，完成安装。

## 登陆 zabbix

安装完成之后，直接跳到登陆界面，使用默认的用户名为“admin”，密码为“zabbix”，登陆 zabbix
![登陆zabbix](http://www.cactifans.org/wp-content/uploads/2015/04/8.png)
登陆之后，会提示 zabbix server 未运行
![登陆之后](http://www.cactifans.org/wp-content/uploads/2015/04/9.png)
启动 zabbix server 和 zabbix agent

```
/usr/local/zabbix/sbin/zabbix_server
/usr/local/zabbix/sbin/zabbix_agentd
```

执行之后，用“ps －ef”，可以看到有如下进程表示启动正常
![进程列表](http://www.cactifans.org/wp-content/uploads/2015/04/10.png)
说明 zabbix server 运行正常
在界面可以看到 zabbix server 的状态已变为“runing”
![服务状态](http://www.cactifans.org/wp-content/uploads/2015/04/11.png)

# 设置 zabbix

## 启用本机监控

默认情况下，没有启用本机监控，我们启用本机监控
![启用本机监控](http://www.cactifans.org/wp-content/uploads/2015/04/12.png)
点击“Disabled”，启用本机监控

## 设置语言

默认语言为英语，设置为中文
![设置语言](http://www.cactifans.org/wp-content/uploads/2015/04/13.png)

## 中文乱码解决

上传字体文件到服务器，我是用微软雅黑的字体，可以搜索下载到，将下载的字体文件 msyh.ttf（16Mb 大小），上传到服务器的/var/www/html/fonts 目录下(也就是 zabbix web 程序根目录下的 fonts 目录下)
编辑 zabbix web 文件

```
vi /var/www/html/include/defines.inc.php
```

96 行，字体名改为 msyh

```
define('ZBX_FONT_NAME', 'msyh');
```

44 行，也改为 msyh

```
define('ZBX_GRAPH_FONT_NAME',           'msyh');
```

保存，刷新界面，看到已经可以正常显示中文了
![界面](http://www.cactifans.org/wp-content/uploads/2015/04/14.png)

# 结语：

本文主要用源码编译的方式安装，也可以用 rpm 包方式安装，可对照官方文档进行操作
[zabbix 2.2 版本官方文档](https://www.zabbix.com/documentation/2.2/manual)
[zabbix 2.4 版本官方文档](https://www.zabbix.com/documentation/2.4/manual)
