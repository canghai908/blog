---
title: Zabbix监控设备SNMP Trap消息
date: 2019-09-27 17:33:13
tags: [zabbix, snmptrap, snmptt]
categories: [zabbix]
---

## 一.SNMP 协议

### 1.协议介绍

snmp 协议是日常使用的较多的一种协议，绝大多数网络设备/存储等都支持 snmp 协议，通过此协议可以实现设备状态的监控及管理。

### 2.主要组成

SNMP 协议包括以下三个部分
![1](https://img.cactifans.com/wp-content/uploads/2019/09/SNMP-7.png)

**SNMP Agent**：负责处理 snmp 请求，主要包括 get，set 等操作。可通过此接口查询设备的运行状态（使用较多），或者变更配置（使用较少），默认使用 UDP 161 端口

**SNMP Trap**： snmp 通知消息，主动发送消息到管理端。如设备故障，端口 down 等都会实时发送消息到接收端。默认使用 UDP 162 端口

**SNMP MIB**：MIB 代表管理信息库，是按层次结构组织的信息的集合,定义了设备内被管理对象的属性。

### 3.基本操作

经常使用的终端命令有以下几个:  
**snmpwalk**：遍历整个 snmp mib 树，通常用来检测 snmp 是否配置成功，一般完整 walk 到一个 MIB，在 MIB 末尾，都会输出“End Of Mib”的字样。否则可能为 Response timeout 等错误，以此来判断 snmp agent 是否配置成功。  
**snmpget**：获取某个 oid 的具体数值，通常用来检测 oid 是否正确。  
**遇到很多使用 snmpwalk 来获取 oid 数据的，这是错误的用法，这样是不能测试 oid 的正确性。只有通过 snmpget+oid 能获取的数据的 oid，才能配置成独立的监控项，作为监控指标采集数据。**  
使用 zabbix 监控 snmp 设备状态，一般按照以下步骤进行操作：

1. 通过 snmpwalk 确定 snmp agent 配置是否正确
2. 通过 snmpget 获取具体的某个 oid 的数据，记录 oid
3. 在 zabbix 上建立对应的 item，interface 选择设备 snmp 接口，Key
   可随意，SNMP OID 输入 oid，SNMP community 输入 snmp community
   Type of information 选择对应类型即可。

## 二.SNMPTrap 监控

### 1.SNMPTT 介绍

SNMPTT (SNMP Trap Translator) 是一个 perl 语言编写的用来处理 snmptrap 消息的程序，可与 Net-SNMP / UCD-SNMP snmptrapd 程序（www.net-snmp.org）一起使用，基本流程如下
![2](https://img.cactifans.com/wp-content/uploads/2019/09/snmptt-flow.jpg)
snmptt 负责处理 net-snmp 接收到的 trap 消息，通过 snmptt 命令或 snmptthandler 进行处理，解析消息之后可将消息写入文件或数据库。

### 2.安装 SNMPTT

操作系统为 centos 7.7 1908 x64，使用 yum 方式安装 snmptt，snmptt 不在默认的官方源，属于 epel 源，因此需要首先安装 epel 源。
添加 epel 源

```
yum install epel-release -y
```

安装 snmptt 及相关组件

```
yum install snmptt perl-Sys-Syslog perl-DBD-MySQL -y
```

即可完成安装。

### 3.转换 MIB 文件

snmp trap 消息如果不翻译,原始内容可能是这样的

```
{
  "Version": 2,
  "TrapType": 0,
  "OID": null,
  "Other": null,
  "Community": "public",
  "Username": "",
  "Address": "172.16.1.64:49692",
  "VarBinds": {
    ".1.3.6.1.2.1.1.3.0": 7908527690000000,
    ".1.3.6.1.2.1.2.2.1.1.18": 18,
    ".1.3.6.1.2.1.2.2.1.2.18": "Vlanif103",
    ".1.3.6.1.2.1.2.2.1.7.18": 2,
    ".1.3.6.1.2.1.2.2.1.8.18": 2,
    ".1.3.6.1.6.3.1.1.4.1.0": [1, 3, 6, 1, 6, 3, 1, 1, 5, 3]
  },
  "VarBindOIDs": [
    ".1.3.6.1.2.1.1.3.0",
    ".1.3.6.1.6.3.1.1.4.1.0",
    ".1.3.6.1.2.1.2.2.1.1.18",
    ".1.3.6.1.2.1.2.2.1.7.18",
    ".1.3.6.1.2.1.2.2.1.8.18",
    ".1.3.6.1.2.1.2.2.1.2.18"
  ]
}
```

如果需要知道上述 oid 的具体内容及数值意义就需要对照 MIB 文件对消息进行“翻译”。snmptt 需要加载对应配置文件，对 trap 消息进行翻译。
snmptt 提供了 snmpttconvertmib 工具，snmpttconvertmib 是一个 Perl 脚本，它将读取 MIB 文件并将 TRAP-TYPE（v1）或 NOTIFICATIONATION-TYPE（v2）定义转换为 SNMPTT 可读的配置文件，从而实现对 trap 消息的翻译，基本命令如下：

```
/usr/bin/snmpttconvertmib --in=/usr/share/snmp/mibs/CPQHOST.mib --out=/etc/snmp/snmptt.conf.compaq --net_snmp_perl
```

- /usr/share/snmp/mibs/ 为 centos7 默认的 mibs 文件路径，上传设备 mib 文件到此目录。
- /etc/snmp/snmptt.cong.compaq 转换完输出的配置文件

由于一般情况设备 mib 可能有多个，建议转换为一个配置文件中，便于管理，可使用以下命令进行批量转换。

```
for i in HUAWEI*
do
/usr/bin/snmpttconvertmib --in=$i --out=/etc/snmp/snmptt.conf.huawei --net_snmp_perl
done
```

以 HUAWEI 开头的 mib 文件都转换为 snmptt.conf.huawei，基本都是一些私有的 MIB。snmptt 自带的 snmptt.conf 配置文件里已经包括了一些常用的配置如端口 up/down。实际过程中，只转换需要关注的 MIB 文件即可。
这里以配置华为 USG 6320 登录 trap 告警为例，从华为官网下载 MIB 文件，并上传到系统/usr/share/snmp/mibs 目录下，从 MIB 说明文件得知，用户登录 MIB 文件属于 HUAWEI-SECURITY-LOGIN-MIB.mib 文件，对 mib 文件进行转换

```
/usr/bin/snmpttconvertmib --in=/usr/share/snmp/mibs/HUAWEI-SECURITY-LOGIN-MIB.mib --out=/etc/snmp/snmptt.conf.HUAWEI-SECURITY-LOGIN --net_snmp_perl
```

输入如下
![3](https://img.cactifans.com/wp-content/uploads/2019/09/61243AC3-F984-4288-9749-1D98D0AD6BA8.jpg?imageView2/1/w/695/h/537#)
表示转换成功，已经生成 snmptt.conf.HUAWEI-SECURITY-LOGIN，**这里转换出来的为标准的文件，不符合 zabbix snmptrap 文件格式，因此还需要执行以下命令对配置文件进行稍加修改。**

```
sed -i 's/FORMAT/FORMAT ZBXTRAP $aA/g' /etc/snmp/snmptt.conf.HUAWEI-SECURITY-LOGIN
```

其实就是添加一个 ZBXTRAP 字符。很多文档遗漏此项，导致后续配置后 zabbix server 日志出现以下错误：

```
28391:20190923:150150.572 invalid trap data found "2019/09/23 15:01:49 .1.3.6.1.6.3.1.1.5.4 Normal "Status Events" 172.16.1.64 - Link up on interface 18.  Admin state: up.  Operational state: up
2019/09/23 15:01:49 .1.3.6.1.6.3.1.1.5.4 Normal "Status Events" 172.16.1.64 - A linkUp trap signifies that the SNMPv2 entity, 18 up up Vlanif103
```

无法识别 trap 文件格式。

### 4.配置 snmptt

snmptt 配置文件有 2 个

- /etc/snmp/snmptt.ini snmptt 主要配置文件
- /etc/snmp/snmptt.conf 系统默认的策略文件，包括一些基本的端口 up/down 的配置
  修改默认的 snmptt.ini 配置文件，主要修改一下几个地方

```
mode = standalone
multiple_event = 0
net_snmp_perl_enable = 1
translate_log_trap_oid = 1
date_time_format = %Y/%m/%d %H:%M:%S
syslog_enable = 0
```

在这里需要加载刚才转换完成的 snmptt.conf.HUAWEI-SECURITY-LOGIN 配置文件
在 snmptt.ini 文件末尾如下添加 snmptt.conf.HUAWEI-SECURITY-LOGIN

```
[TrapFiles]
# A list of snmptt.conf files (this is NOT the snmptrapd.conf file).  The COMPLETE path
# and filename.  Ex: '/etc/snmp/snmptt.conf'
snmptt_conf_files = <<END
/etc/snmp/snmptt.conf
/etc/snmp/snmptt.conf.HUAWEI-SECURITY-LOGIN
END
```

snmptt 有二种模式，分别为

- standalone 每次收到 trap 消息时读取 snmptt.ini 配置文件调用 snmptthandler 来处理 trap 消息
- daemon 顾名思义守护进程方式，启动 snmptrap 时，初始化时一次性读取 snmptt.ini
  二种方式各有区别，选择使用即可，具体区别可查看官方说明[snmptt mode](http://snmptt.sourceforge.net/docs/snmptt.shtml#Installation-Unix)

由于这里测试使用，经常修改 snmptt.ini 配置文件，如果使用 daemon 模式，那么每次修改 snmptt.ini 配置文件就需要重启 snmptt，因此这里我使用 standalone 模式。

### 5.配置 snmptrap

snmp trap 消息为主动通知，因此需要配置服务器来接收设备发送过来的 snmp trap 消息。net snmp 接收 trap 消息后，通过 traphandle 调用 snmptt 来对 trap 消息进行处理。
安装 net snmp，建议安装 net-snmp 的所有关联包

```
yum install net-snmp* -y
```

即可完成 snmp 安装。
修改/etc/sysconfig/snmptrapd 为以下内容

```
OPTIONS="-m +ALL -On"
```

修改/etc/snmp/snmptrapd.conf 配置文件,添加如下

```
authCommunity execute public
traphandle default /usr/sbin/snmptt
```

authCommunity 可以配置多个
启动 snmptrap 服务

```
systemctl start snmptrapd
systemctl enable snmptrapd
```

在 usg 6320 上配置 trap 消息，并指向 snmp trap server，配置 Community 为 public
![4](https://img.cactifans.com/wp-content/uploads/2019/09/D283777A-9467-446F-84A0-96D969F38B71.jpg)
配置之后，通过 web 登录 USG 和退出 USG，都会收到 trap 消息。snmptt 日志文件为/var/log/snmptt/snmptt.log 即可看到
![6](https://img.cactifans.com/wp-content/uploads/2019/09/42B6BB7E-74F6-4DDC-8545-58702BD680E3.jpg)
表示配置处理 trap 成功，原始内容都为变量内容，不太友好，
可以通过调整 snmptt.conf.HUAWEI-SECURITY-LOGIN，配置 trap 消息输出格式。
snmptt.conf.HUAWEI-SECURITY-LOGIN 配置文件内容大致如下

```
...
EVENT hwSecLOGINSucced .1.3.6.1.4.1.2011.6.122.62.2.1 "Status Events" Normal
FORMAT ZBXTRAP $aA $*
SDESC

The user login succeeded.
Variables:
  1: hwSecLOGINUser
     Syntax="OCTETSTR"
     Descr="The user name."
  2: hwSecLOGINIP
     Syntax="OCTETSTR"
     Descr="The User IP address."
  3: hwSecLOGINTime
     Syntax="OCTETSTR"
     Descr="The User login time."
  4: hwSecLOGINType
     Syntax="OCTETSTR"
     Descr="The User access type."
  5: hwSecLOGINLevel
     Syntax="INTEGER32"
     Descr="The User login level."
EDESC
......
```

其中 FROMAT 就是需要修改定义的，其中包括很多变量，具体可在[SNMPTT.CONF-FORMAT](http://snmptt.sourceforge.net/docs/snmptt.shtml#SNMPTT.CONF-FORMAT) 常用的几个如下：

- \$aA snmp trap 的代理 IP，即源 IP
- \$\* 所有变量
- \$n n 为数字，表示变量次序
  Variables 为 trap 变量，基本可以看懂
- hwSecLOGINUser 登录用户名
- hwSecLOGINIP 登录 IP
- hwSecLOGINTime 登录时间
- hwSecLOGINType 登录类型
- hwSecLOGINLevel 登录级别
  按照变量，可以对 trap 消息进行定制化，还支持中文！将 FORMAT 一行修改为

```
FORMAT ZBXTRAP $aA User login succeed！userName = $1, loginIP = $2, loginTime = $3, accessType = $4, userLevel = $5
```

修改后，登录 USG，查看 snmptt.log 日志,日志已经被格式成需要的格式，已经可以看懂了！

```
2019/09/27 15:08:49 hwSecLOGINSucced Normal "Status Events" 172.16.1.64 - ZBXTRAP 172.16.1.64 User login succeed!userName = admin, loginIP = 10.128.23.19, loginTime = 2019/09/27 16:54:08, accessType = web, userLevel = 15
```

至此完成了 snmp trap 消息的接收及翻译工作。

## 三.配置 Zabbix

### 1.zabbix 安装注意事项

zabbix 可以配置读取接收到的 trap 文件，从而实现对对 trap 消息的告警，如过 zabbix 是编译安装，系统需要安装 net-snmp-devel 包，编译时需要添加以下参数

```
–with-net-snmp
```

### 2.修改 zabbix server 配置文件

修改 zabbix server 配置文件

```
SNMPTrapperFile=/var/log/snmptt/snmptt.log
StartSNMPTrapper=1
```

制定 trap 文件为 snmptt 日志文件，并开启 strap 处理的 poller，修改之后重启 zabbix server。

### 3.配置 snmptrap item

添加一个主机
![11](https://img.cactifans.com/wp-content/uploads/2019/09/2A71642A-B1B7-4343-BBB5-200CED67B565.jpg)
SNMP interfaces 添加机器 Ip，添加之后创建 item
![12](https://img.cactifans.com/wp-content/uploads/2019/09/6AD29BE7-D0D9-4ACF-8129-8B57FA6D53FA.jpg)

- Type 为 SNMP trap
- snmptrap 为 zabbix 内置的 key，用来获取 trap 文件内容
- Type of information 选择 log
- Log time format 和 snmptt 配置一致，配置为 yyyy/MM/dd hh:mm:ss
  直接配置 Key 为 snmptrap，表示获取 snmptt 日志里的所有消息。配置之后登录设备，退出设备，可在 Last data 看到数据已经采集。（此截图为为进行翻译之前截取，翻译之后可显示中文，英文）
  ![13](https://img.cactifans.com/wp-content/uploads/2019/09/405CDF74-6931-46FD-8D06-0DB8F4BAE297.jpg)

### 4.配置 Trigger

目前已经采集了 trap 日志，需要配置对应的 trigger 对 trap 消息进行告警。以 usg 设备登录错误为例，登录错误时间类型为:hwSecLOGINFailed,因此配置如下 trigger
![14](https://img.cactifans.com/wp-content/uploads/2019/09/000E463B-F0C4-48EE-9F92-2D44AA04DB58.jpg)
配置之后，用错误的用户登录 USG，发现已经告警了。
![15](https://img.cactifans.com/wp-content/uploads/2019/09/9A98EEEA-4C1B-4214-8DA6-36DF8C4B5A64.jpg)
由于 zabbix 配置了微信告警，微信上可以看到如下消息：
![16](https://img.cactifans.com/wp-content/uploads/2019/09/WechatIMG99.png?imageView2/1/w/577/h/1024#)
告警内容有点模糊，同样修改下 snmptt.conf.HUAWEI-SECURITY-LOGIN,将登录失败的内容修改为容易理解的中文，即修改 FORMAT 为中文

```
EVENT hwSecLOGINFailed .1.3.6.1.4.1.2011.6.122.62.2.2 "Status Events" Normal
FORMAT ZBXTRAP $aA 用户登录设备失败!用户名 = $1,登录ip = $2, 登录时间 = $3, 接入类型 = $4, 用户级别 = $5
SDESC

The user login failed.
Variables:
  1: hwSecLOGINUser
     Syntax="OCTETSTR"
     Descr="The user name."
  2: hwSecLOGINIP
     Syntax="OCTETSTR"
     Descr="The User IP address."
  3: hwSecLOGINTime
     Syntax="OCTETSTR"
     Descr="The User login time."
  4: hwSecLOGINType
     Syntax="OCTETSTR"
     Descr="The User access type."
EDESC
```

配置后告警内容
![12](https://img.cactifans.com/wp-content/uploads/2019/09/095EAD87-D08C-4FC1-ADBB-F9442EA026C6.jpg)
5 分钟后故障自动恢复，至此配置完成。

### 5.一般用法

以上只是简单测试，生产环境中应该注意以下事项

- 分析需要告警的 trap event 类型
  如:hwSecLOGINFailed/warmStart/linkDown/coldStart/authenticationFailure 根据类型，配置对应的 triger 即可
- 可直接配置对应的 key 来采集对应的指标,只要收到消息就告警。如下图配置只采集 hwSecLOGINSucced 的指标。
  ![23](https://img.cactifans.com/wp-content/uploads/2019/09/D87546C7-05C7-43E6-877D-C79E11807537.jpg)
  配置对应的 trigger，进行告警
  ![25](https://img.cactifans.com/wp-content/uploads/2019/09/B51753F4-F986-4949-8A19-07D39B327484.jpg)
- 如只配置了采集对应的 item，而没有配置 snmptrapkey，建议配置一个 snmptrap.fallback 的 item，配有匹配的 item 消息，都会保存到此 item 里
- 更多配置及用法参考 zabbix 官网配置[zabbix trap](https://zabbix.org/wiki/Start_with_SNMP_traps_in_Zabbix)

## 四.注意事项

### 1.MIB 文件

一定要确保 MIb 文件的准确性，设备版本与 MIB 文件版本必须一致，包括大小版本。如不一致会看到解析文件出现以类似错误

```bash
2019/09/27 08:34:23 .1.3.6.1.4.1.2011.6.122.62.2.2 Normal "Status Events" 172.16.1.64 - the user 123 login with web on 10.128.23.19 in 2019/09/27 08:34:22 by time on 2019/09/27 08:34:22 ,status is Wrong Type (should be OCTET STRING): 0
```

说明 mib 文件里定义的 trap 变量类型和实际收到的 trap 消息变量类型不一致，解析错误。

### 2.zabbix 配置

建议对同一类型设备配置统一的 snmp trap 模版，相同型号直接关联模版即可使用。

### 3.snmptt 日志

随之 snmp trap 消息的增多，snmptt 会逐渐变大，建议配置 logrotate 对日志进行切分,使用 yum 安装的 snmptt 已经自动配置了，使用编译安装的，可以参考
/etc/logrotate.d/snmptt

```
/var/log/snmptt/snmptt*.log /var/log/snmptt/snm ptthandler.debug {
    weekly
    notifempty
    missingok
}
/var/log/snmptt/snmptt.debug {
    weekly
    notifempty
    missingok
    postrotate
        /etc/init.d/snmptt reload >/dev/null 2>/dev/null || true
    endscript
}
```

snmptt 可开启 debug 日志，建议在测试阶段开启 debug 日志，以便进行排错。

以上为 snmp trap 告警的全面解读，下期预告:《基于 Web 方式的 snmp trap 管理》敬请关注！

**参考文档**  
1.http://snmptt.sourceforge.net/docs/snmptt.shtml  
2.https://www.miraclelinux.com/product-service/zabbix/tech-lounge/zbx-tl-001

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
