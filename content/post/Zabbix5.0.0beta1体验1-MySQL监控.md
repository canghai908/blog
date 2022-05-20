---
title: "Zabbix 5.0.0beta1体验1- MySQL监控"
date: 2020-04-24T13:59:05+08:00
tags: [zabbix, agent2]
categories: [zabbix]
---

距离上次发布文章已经有几个月时间，最近复工事情较多，耽误了一段时间。回头一看 zabbix5.0beta1 版本已经发布了！下面介绍一下新版本的功能。
**5.0 目前并未正式发布，切勿使用在生产环境！！！**

## 一.新功能介绍

### 1.界面展示细化

5.0 版本开始调整为左侧菜单，配置和查看的部分从功能上进行了分离,更加直观
![1](https://img.cactifans.com/wp-content/uploads/2020/04/CDA76F30-3742-47EA-96E2-912FE30A78EB.png)
Monitoring 菜单下 Hosts，可直观看到主机的状态
![1](https://img.cactifans.com/wp-content/uploads/2020/04/65E3CBAC-E36B-4962-9A8A-199A65622FFA.png)
Configureation 菜单下可对主机等资源进行配置操作
![1](https://img.cactifans.com/wp-content/uploads/2020/04/D6688274-5187-49E3-B284-D4EB9D44C220.png)
图形查看
![1](https://img.cactifans.com/wp-content/uploads/2020/04/00B3FB21-7E68-46F2-9F28-FF78C4EDE58E.png)

### 2.Agent2 集成插件

Agent2 使用 go 语言编写，其中已经内置了很多组件的监控
官方还发布了插件的开发教程，二次开发更加灵活
其他功能太多，不便于一一解释，后期再给大家详细讲解。
![1](https://img.cactifans.com/wp-content/uploads/2020/04/895E321B-16EF-4185-ABA2-92DC5A06B69D.png)
看完上面新功能，下面为大家介绍新版本的实用功能 MySQL 监控

## 二.Mysql 监控

MySQL 作为使用最广泛的数据库，使用 Zabbix 监控 MySQL 一直是一个长期的话题，教程更是多如牛毛，之前我也写过一个监控 MySQL 的小组件[Zabbix 监控 MySQL](https://blog.cactifans.com/2018/08/08/Zabbix%E7%9B%91%E6%8E%A7MySQL%E5%B7%A5%E5%85%B7/) Zabbix5.0 已经内置了 MySQL 监控，使用极为方便。

### 1.安装 Zabbix Agent2

Zabbix Server 安装和之前并无区别，直接编译安装即可
目前阶段,使用源码编译安装 Agent2，编译需要 gcc，go 语言环境。
配置 go 语言环境,以 centos7.6 操作系统为例子
下载 go 安装包

```
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz
tar zxvf go1.14.2.linux-amd64.tar.gz -C /usr/local/
```

配置环境变量，修改/etc/profile 末尾添加

```
export GOROOT=/usr/local/go
export GOPATH=/home/mygo
export PATH=$PATH:/usr/local/go/bin:/usr/local/zabbix/bin:/usr/local/zabbix/sbin
```

使环境变量生效

```
source /etc/profile
```

配置 GoProxy

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

下载 Zabbix 5.0.0beta1 源码并编译安装,安装过程需要联网下载依赖包

```
wget https://cdn.zabbix.com/development/5.0.0beta1/zabbix-5.0.0beta1.tar.gz
tar zxvf zabbix-5.0.0beta1.tar.gz
cd  zabbix-5.0.0beta1
./configure --prefix=/usr/local/zabbix --enable-agent2
make
make install
```

### 配置安装 Zabbix Agent2

配置与之前的 Agent 的配置没有太大区别，基本通用
修改配置文件 zabbix_agent2.conf

```
Server=172.16.66.9
ServerActive=172.16.66.9
Hostname=jenkins43
```

172.16.66.9 为 Zabbix ServerIP
Hostname 可配置为服务器主机名
使用以下命令启动 Agent2

```
 nohup /usr/local/zabbix/sbin/zabbix_agent2 -c /usr/local/etc/zabbix_agent2.conf  > myout.file 2>&1 &
```

Agent2 可安装在被监控 MySQL 主机上，也可以安装在其他机器，使用远程连接 MySQL 的方法进行监控，本次使用远程连接方式实现,环境如下

|     主机     | 安装组件      |
| :----------: | :------------ |
| 172.16.66.43 | Zabbix Agent2 |
| 172.16.66.9  | MySQL         |

使用以下命令在被监控 MySQL 里创建独立监控用户并授权远程访问，避免使用业务用户

```
create user mon@'172.16.66.43' identified by 'monpwd123';
```

Agent2 没有依赖包，可直接拷贝二进制文件 zabbix_agent2 和配置文件 zabbix_agent2.conf 到目标机器，即可完成安装

### Zabbix Server 配置

在 Zabbix Server 上添加主机，
![1](https://img.cactifans.com/wp-content/uploads/2020/04/D967B22B-3A89-44B3-B488-32E37A921F45.png)
并关联 MySQL 模版，注意使 By Agent2 模版
![1](https://img.cactifans.com/wp-content/uploads/2020/04/E7B6CA17-615B-4E54-852B-49B5ADFC9B99.png)
并添加以下三个宏变量

```
{$MYSQL.DSN}    	mysql的连接串，可使用TCP和Unix
tcp://myhost 或 unix:/var/run/mysql.sock
{$MYSQL.USER}   	mysql用户
{$MYSQL.PASSWORD}   对应的用户密码
```

这里这配置刚才建立好的 MySQL 用户信息
![1](https://img.cactifans.com/wp-content/uploads/2020/04/A4C8D1C8-883B-43E1-90F2-CFDA8D6A14B8.png)

### 效果

Last Data
![1](https://img.cactifans.com/wp-content/uploads/2020/04/787B589C-A988-46E4-812A-F9733DC478D3.png)
图形 1
![1](https://img.cactifans.com/wp-content/uploads/2020/04/6E193FF3-DE6B-494B-A4A2-45FCE4BB4D85.png)
图形 2
![1](https://img.cactifans.com/wp-content/uploads/2020/04/675477D6-627C-4BD6-B003-B770225C4D49.png)

### 结语

Zabbix 5.0 还有众多功能，配置和监控更加简单灵活。使用 Agent2 使用 go 编写，减低了二次开发的门槛，后续会发布插件开发文章，期待正式版本发布。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
