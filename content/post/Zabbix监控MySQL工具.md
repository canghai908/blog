---
title: Zabbix监控MySQL工具
date: 2018-08-08 17:15:24
tags: [zabbix, mysql, trapper]
categories: [zabbix]
---

## 介绍

最近学习使用 go 语言写了一个 zabbix 监控 mysql 数据库的小工具，有如下特点:  
1.使用 Zabbix Agent Trapper 方式(主动发送采集数据到 zabbix server，类似 active 模式)监控 mysql 数据库  
2.支持对密码加密，避免配置文件里出现明文密码  
3.支持 SHOW /_!50001 GLOBAL _/ STATUS 和 SHOW /_!50001 GLOBAL _/ VARIABLES 所有指标监控！！！  
4.支持 mysql 主从监控(默认关闭，可通过配置文件开启，mysql 用户需要有 SUPER 或 REPLICATION CLIENT 权限)  
5.支持自定义采集周期

源码：https://github.com/canghai908/zabbix-mymon
新手上路,轻喷！欢迎 star！
模版下载:[https://dl.cactifans.com/zabbix/zabbix_template_mysql.tar.gz](https://dl.cactifans.com/zabbix/zabbix_template_mysql.tar.gz)  
工具下载:[https://dl.cactifans.com/zabbix/zabbix-mymon-0.0.1.x86_64.tar.gz](https://dl.cactifans.com/zabbix/zabbix-mymon-0.0.1.x86_64.tar.gz)

## 源码编译

如果不喜欢使用二进制包或者需要修改某些源码的，可以使用源码编译，具体步骤如下。部署好 golang 开发环境，具体部署手册请查看 https://golang.org/doc/install
执行以下命令,即可编译生成 mymon-0.0.1.tar.gz

```
mkdir -p $GOPATH/src/github.com/canghai908
cd $GOPATH/src/github.com/canghai90
git clone https://github.com/canghai908/zabbix-mymon.git
cd zabbix-mymon &./control pack
```

## 导入模版

在 zabbix Server 上导入导入模版，解压之前下载的模版。
先导入 valuemap，导入 zbx_valuemaps_mysql.xml
![1](https://img.cactifans.com/wp-content/uploads/2018/08/1.jpg)
再导入模版文件，zbx_templates_mysql.xml
![2](https://img.cactifans.com/wp-content/uploads/2018/08/2.jpg)
导入之后可以看到名为 Template App MySQL Trapper 的模版，表示导入成功
![3](https://img.cactifans.com/wp-content/uploads/2018/08/3.jpg)
MySQL 作为中间件可以挂载到任何在 zabbix server 里的 host 上。监控脚本不一定部署在真实的数据库服务器之上，只要脚本通过远程方式能连接到数据库即可。
关联模版到需要挂载 Mysql 监控的的 host 上即可。

## 配置插件

下载并解压插件

```
mkdir -p  /opt/mymon
wget https://dl.cactifans.com/zabbix/zabbix-mymon-0.0.1.x86_64.tar.gz
tar zxvf zabbix-mymon-0.0.1.x86_64.tar.gz -C /opt/mymon
```

插件结构  
├── control //启动脚本  
├── mymon //二进制程序  
└── mymon.json //配置文件  
使用 mysql 的 root 用户进行监控（主从监控需要）。把密码写在明文的文件里不是被推荐的，因此脚本提供了一个使用 AES 加密算法加密数据库密码的工具，保证 root 密码的安全。使用一下命令加密密码明文，将 yourpassword 替换为你的 root 密码

```bash
/opt/mymon/mymon enc yourpassword
```

执行之后会看到进过加密后的密码密文,记录下来

```bash
/opt/mymon/mymon enc admin
sXcEQ2FTGk4WsWSxyT6fuBnjZ3v43pc0
```

修改配置文件 mymon.json

```
{
"debug": false,
"interval": 60,
"slave": false,
"mysql": {
        "username": "admin",
        "password": "hcxhF+KoURUsge+kMQQaU2lDN1YfOLiJ",
        "host": "172.16.66.17",
        "port": 3306
    },
"zabbix":{
        "server": "zabbix.cactifans.com",
        "port": 10051,
        "hostname": "hosts135"
    }
}
```

配置文件说明  
interval 采集周期，单位为秒  
slave 是否开启 slave 采集,如需要采集,mysql 用户需要有 SUPER 或 REPLICATION CLIENT 权限  
需要监控的 mysql 数据库信息配置

> username 数据库的用户名，一般使用 root 用户  
> passoword 加密后的密码密文  
> host 数据库主机 ip  
> port mysql 端口

zabbix 信息配置

> server 为 zabbix server 的地址,如通过 zabbix proxy 需要设置为 zabbix proxy 的地址  
> port zabbix server 端口默认为 10051  
> hostname 为之前关联模版的主机名一致

![4](https://img.cactifans.com/wp-content/uploads/2018/08/4.jpg)

## 使用

如不需要采集 slave 信息,用以下命令建立一个 mysql 用户不用授权任何权限即可监控

```sql
CREATE USER admin@'%' IDENTIFIED BY 'password';
```

如需要监控 slave 信息，需要将配置文件里的 slave 设置为 true，并建立一个有 SUPER 或 REPLICATION CLIENT 权限的用户,修改好配置文件之后可以启动插件,使用以下命令进行测试数据库是否能够连通

```
cd /opt/mymon
./mymon ping
```

可以看到使用的配置文件  
如返回 1，表示数据库连接正常  
如返回 2 表示连接数据库异常，请检查用户权限及配置文件

```bash
2018/08/08 15:29:58 ping.go:41: Using config file: /opt/mymon/mymon.json  successfully!
1
```

测试成功之后可以使用以下命令启动即可

```bash
./control start
```

常用操作

```
./control start    //启动应用
./control stop     //停止应用
./control restart  //重启应用
./control tail     //查看日志
```

## 效果

![5](https://img.cactifans.com/wp-content/uploads/2018/08/5.jpg)
![6](https://img.cactifans.com/wp-content/uploads/2018/08/6.jpg)
![7](https://img.cactifans.com/wp-content/uploads/2018/08/7.jpg)
![8](https://img.cactifans.com/wp-content/uploads/2018/08/8.jpg)

## 扩展

### 指标增加

由于指标较多目前添加了基础的监控指标，SHOW /_!50001 GLOBAL _/ STATUS 和 SHOW /_!50001 GLOBAL _/ 命令支持的指标都支持监控！！！只需要在模版里添加新的 item 即可。clone 当前的指标,修改就可以了  
![9](https://img.cactifans.com/wp-content/uploads/2018/08/9.jpg)

指标解释

> name 为指标名称  
> type 不修改，为 Zabbix trapper  
> key 为 myql.加上 SHOW /_!50001 GLOBAL _/ STATUS 和 SHOW /_!50001 GLOBAL _/ 命令里的指标名称  
> type of Information 为指标类型，根据具体指标类型选择  
> preprocessing 指标是计数器还是具体数值具体设置即可

### 命令行工具

工具内置几个命令行工具及基本使用,可以使用 mymon -h 查看帮助

```bash
[root@localhost mymon]# ./mymon
Zabbix mysql database monitoring tool. For example:

mymon daemon --config=./mymon.json

Usage:
  mymon [command]

Available Commands:
  daemon      Running as a daemon
  enc         Encrypt passwords in AES mode
  help        Help about any command
  ping        Connected line checker
  version     Version

Flags:
      --config string   config file (default is /.mymon.json)
  -h, --help            help for mymon

Use "mymon [command] --help" for more information about a command.
```

## 注意事项

1.trapper 方式默认允许任何主机发送数据到 zabbix server，建议通过设置宏的方式，在模版里配置 allowed hosts 配置权限
2.mysql 是否运行状态未监控，建议添加 mysql 进程监控来实现

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
