---
title: "monitoring tomcat with zabbix"
date: 2015-07-24 10:21:36
tags:
  - zabbix
  - tomat
  - jmx
categories: [zabbix]
---

zabbix 提供了一个 java gateway 的应用去监控 jmx（Java Management Extensions，即 Java 管理扩展）是一个为应用程序、设备、系统等植入管理功能的框架。JMX 可以跨越一系列异构操作系统平台、系统体系结构和网络传输协议，灵活的开发无缝集成的系统、网络和服务管理应用。

## 一. Zabbix 的 JMX 监控架构

![1](https://img.cactifans.com/wp-content/uploads/2015/07/01.jpg)

## 二.安装 Java gateway［zabbix 服务端操作］

### 1.安装 jdk

首先要安装 jdk，我的系统位 redhat 6.4 x64 位，我使用 rpm 包安装 jdk，从http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html下载最新的jdk，rpm包（我使用7u79版本，最新为8）并上传到zabbix server

```
rpm －ivh jdk-7u79-linux-x64.rpm
```

安装成功之后添加系统环境变量

```
vi /etc/profile
```

添加如下

```
JAVA_HOME=/usr/java/jdk1.7.0_79
JRE_HOME=/usr/java/jdk1.7.0_79/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

使配置生效

```
source /etc/profile
```

测试，输入

```
java -version
```

如果显示如下信息表示 jdk 安装完成

```
java version "1.7.0_79"
Java(TM) SE Runtime Environment (build 1.7.0_79-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.79-b02, mixed mode)
You have mail in /var/spool/mail/root
```

### 2.安装 Java gateway

可使用 rpm 进行安装，我使用源码安装，大家可在安装 zabbix 时启用--enable-java 参数即可安装 zabbix java gateway，如果第一次没有加载，可重新加载编译安装，我使用以下参数安装

```
./configure --prefix=/usr/local/zabbix --enable-server --enable-agent --enable-java --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2
```

以上参数可安装 zabbix server，java gateway，snmp，zabbix agent
，建议大家使用。

### 3.配置 Java gateway

安装好之后可在/usr/local/zabbix/sbin/目录下看到 zabbix_java 目录（具体根据实际安装情况）
编辑配置文件

```
vi /usr/local/zabbix/sbin/zabbix_java/settings.sh
LISTEN_IP="127.0.0.1"
LISTEN_PORT=10052
PID_FILE="/tmp/zabbix_java.pid"
START_POLLERS=5
```

修改 zabbix server 配置文件

```
vi /usr/local/zabbix/etc/zabbix_server.conf
JavaGateway=127.0.0.1
JavaGatewayPort=10052
StartJavaPollers=5
```

### 4.启动测试 Java gateway

启动 Java gateway

```
/usr/local/zabbix/sbin/zabbix_java/startup.sh
```

可使用

```
netstat -lntp | grep 10052
```

查看是否已经监听 10052 端口，如果已监听，表示启动成功，如果没有，可通过 zabbix_server 日志查看解决.

### 5.添加 catalina-jmx-remote.jar

添加 catalina-jmx-remote.jar 到 zabbix java gateway 的 lib 目录下，catalina-jmx-remote.jar 包可在http://archive.apache.org/dist/tomcat/下，在各版本目录的bin/extras/子目录下

```
cd /usr/local/zabbix/sbin/zabbix_java/lib/
wget http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.61/bin/extras/catalina-jmx-remote.jar
```

并重启 zabbix 和 java gateway

```
/usr/local/zabbix/sbin/zabbix_java/shutdown.sh
/usr/local/zabbix/sbin/zabbix_java/startup.sh
/etc/init.d/zabbix-server restart
```

### 6.下载测试工具 cmdline-jmxclient-0.10.3.jar

cmdline-jmxclient-0.10.3.jar 为一个测试工具，可用来测试 jmx 是否配置正确，下载 cmdline-jmxclient-0.10.3.jar(下载到任意目录)

```
wget http://crawler.archive.org/cmdline-jmxclient/cmdline-jmxclient-0.10.3.jar
```

## 三.被监控 tomcat 配置［被监控 tomcat 操作］

### 1.添加 catalina-jmx-remote.jar

添加 catalina-jmx-remote.jar 文件到 tomcat 的 lib 目录

```
cd /opt/apache-tomcat/lib/
wget http://archive.apache.org/dist/tomcat/tomcat-7/v7.0.61/bin/extras/catalina-jmx-remote.jar
```

### 2.修改 setenv.sh

```
vi /opt/apache-tomcat/bin/setenv.sh
```

添加如下

```
CATALINA_OPTS="${CATALINA_OPTS} -Djava.rmi.server.hostname=192.168.7.186"
CATALINA_OPTS="${CATALINA_OPTS} -Djavax.management.builder.initial="
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote=true"
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote.ssl=false"
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote.authenticate=false"
```

注意：hostname 位机器 ip 地址，即被监控机器对外服务地址

### 3.修改 server.xml

```
vi /opt/apache-tomcat/conf/server.xml
```

添加如下，注意添加位置在

```
 <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
```

之后添加如下代码

```
  <Listener className="org.apache.catalina.mbeans.JmxRemoteLifecycleListener" rmiRegistryPortPlatform="12345" rmiServerPortPlatform="12346" />
```

### 4.添加 tomcat 应用自动发现脚本

由于默认的 jmx 不能自动发现部署的新应用，因此需要添加一个脚本自动发现，并且利用 zabbix 的 LLD（Low-level discovery）来发现

```
mkdir /opt/tomcat/
mkdir /tmp/tomcat/
chmod -R 777 /tmp/tomcat/
cd /opt/tomcat/
wget https://dl.cactifans.com/tools/context.sh
wget https://dl.cactifans.com/tools/cmdline-jmxclient-0.10.3.jar
chmod a+x /opt/tomcat/context.sh
echo "*/1  *  *  *  * root /opt/tomcat/context.sh" >>/etc/crontab
```

context.sh 脚本内容

```
#!/bin/bash
java -jar /opt/tomcat/cmdline-jmxclient-0.10.3.jar - 192.168.7.186:12345 Catalina:type=Manager,*| awk -F "," '{ print $1 }'|awk -F: '{ print $2 }'>/tmp/tomcat/context.csv
```

此脚本在被监控端添加，注意要把 ip 改成本机的服务 IP 地址
添加一个应用自动发现的程序，我放到 zabbix agent 的 bin 目录下

```
cd /usr/local/zabbix/bin/
wget https://dl.cactifans.com/tools/context
chmod a+x /usr/local/zabbix/bin/context
```

修改 zabbix agentd 设置，添加应用自动发现的 key

```
vi /usr/local/zabbix/etc/zabbix_agentd.conf
#tomcat
UserParameter=tomcat.context.discovery,/usr/local/zabbix/bin／context
```

重启 tomcat

```
/opt/apache-tomcat/bin/shutdown.sh
/opt/apache-tomcat/bin/startup.sh
```

重启 zabbix agent

```
service zabbix-agent restart
```

注：防火墙需要开放 12345，12346 端口

## 四.测试并添加

### 1.测试

在 zabbix server 上执行

```
java -jar /root/cmdline-jmxclient-0.10.3.jar - 192.168.7.186:12345 java.lang:type=Memory NonHeapMemoryUsage
```

如果有如下回显表示 jmx 配置正确，如不正确，请检查配置

```
07/23/2015 23:25:29 +0800 org.archive.jmx.Client NonHeapMemoryUsage:
committed: 271646720
init: 270991360
max: 587202560
used: 38207984
```

在 zabbix server 上用 zabbix_get 执行

```
zabbix_get -s 192.168.7.153 -k tomcat.context.discovery
```

如果能回显如下数据表示获取成功

```json
{
  "data": [
    {
      "{#CONTEXT}": "context=/manager"
    },
    {
      "{#CONTEXT}": "context=/omai"
    },
    {
      "{#CONTEXT}": "context=/examples"
    },
    {
      "{#CONTEXT}": "context=/docs"
    },
    {
      "{#CONTEXT}": "context=/"
    },
    {
      "{#CONTEXT}": "context=/host-manager"
    }
  ]
}
```

### 2.添加模版

模版我已经做好了大家可直接倒入使用
模版下载：[zabbix_tomcat_templates.xml](https://dl.cactifans.com/tools/zabbix_tomcat_templates.tar.gz)（2016 年 10 月 24 日新地址已更新！）
导入到 zabbix，并关联到主机

## 五.最终效果

![enter image description here](https://img.cactifans.com/wp-content/uploads/2015/07/02-1024x594.jpg)
