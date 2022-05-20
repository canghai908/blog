---
title: zabbix-3.4新功能介绍1——Preprocessing
date: 2017-12-29 01:14:50
tags: [zabbix, Preprocessing, nginx监控, 新功能]
categories: [zabbix]
---

随着 3.4 版本的发布,出现了一大波新功能，后续会陆续推出 3.4 版本新功能介绍及实践.本次说一下 3.4 新增的 Preprocessing 这个功能.（3.4 中文翻译好像有点问题把 Preprocessing 翻译为进程，翻译有点错误）Preprocessing 为预处理，预加工（google 翻译^\_^）使用这个功能可以对 item 收到的数据行处理，处理之后再存入数据库或展示出来.
下面结合一个监控 nginx 状态的实际应用来介绍一下 item 预处理功能及 Dependent item 的使用.

## Nginx status 配置

nginx 作为一款强大的 web 服务器已被广泛使用，结合[nginx-module-vts](https://github.com/vozlt/nginx-module-vts)插件可以将 nginx 的状态通过 http 方式输出，可使用这种方式来监控 nginx 的运行状态，配置好插件之后,访问插件页面可以看到如下状态页面

![status](https://img.cactifans.com/wp-content/uploads/2017/12/913CF9A8-5A4F-45BF-9F3D-78CBF37887E3.jpg)
点击下面的 json 可以输出为标准的 json 格式
![json](https://img.cactifans.com/wp-content/uploads/2017/12/4FA6EBB7-55F9-48AA-9C4E-75FE212AD7FE.jpg)
表明 nginx 配置完成

## Zabbix agent 配置

zabbix 自带 web 监控,由于功能较弱,因此我自己写了一个类似 web 监控的工具，通过 GET 方式访问 nginx status 页面（任何 GET 请求都可），将返回结果输出.web_get 我自己写的 web 检测小工具,大家可下载使用.
web_get 工具下载:
Linux 64 位 https://dl.cactifans.com/tools/web_get.x86_64.tar.gz
Windows 64 位 https://dl.cactifans.com/tools/web_get.windows_amd64.zip
配置 zabbi 修改 zabbix agent 配置文件,添加如下（根据自己实际路径配置，并赋予可执行权限）

```bash
UserParameter=nginx.status[*],/usr/local/zabbix/share/zabbix/web_get $1 $2
```

配置之后重启 zabbix agent.可在 zabbix server 上使用 zabbix_get 命令进行测试 UserParameter 是否工作正常

```bash
zabbix_get -s 192.168.0.1 -k nginx.status[http,exp.test.com/ngx_status/format/json]
```

**192.168.0.1**为被监控客户端，其中包含 2 个参数，第一个**http**为协议，一般为 http 或 https，后面为要监控的**nginx status**的具体 URL.根据自身具体情况配置.如果返回如下信息，表示配置成功
![json1](https://img.cactifans.com/wp-content/uploads/2017/12/C80D64F9-C196-44C4-9E91-EA67D268AC1F.jpg)

## Zabbix server 配置

上一步在 zabbix agent 配置文件中指定了 key 为 nginx.status,因此这里建立的第一个 item 的 key 应该为 nginx.status.这里可以建一个模版,把这些 key 统一到一个模版,我这里为了演示就直接在 host 上建立 item，如图
![status](https://img.cactifans.com/wp-content/uploads/2017/12/18AFE491-CB7E-4F32-B7EB-8104DB702190.jpg)
type 为 zabbix agent,这里 key 里有 2 个参数，第一个参数为协议，为 http 或 https,第二个参数为 nginx status 的 URL 地址.由于返回结果为 json，因此 Type of information 选择**Text**即可.建立**Master item**之后,开始建立[Dependent item](https://www.zabbix.com/documentation/3.4/manual/config/items/itemtypes/dependent_items?s[]=dependent&s[]=item)这也是 3.4 的新功能之一，大致为这些 item 的数据都是来自 Master item.通过 Preprocessing 处理 master item 的数据而获得.建立之前先使用 zabbix_get 获取 json 数据，复制 json 字符串,并使用[在线工具](http://tool.oschina.net/codeformat/json)或编辑器格式化 json,可看到 json 的层次.
![fmt](https://img.cactifans.com/wp-content/uploads/2017/12/3C6486A2-4FC8-4D36-B255-C11D2B4812E3.jpg)
如果我们要获取 json 中 nginxVersion 这个数据，我们可以这样建立一个 Dependent item.点击 create item
![status1](https://img.cactifans.com/wp-content/uploads/2017/12/D0C0FFFC-8D32-47F6-9116-D1F8D712625A.jpg)
item name 可随便填写，能标示就成.type 这里一定要选择 **Dependent item**，key 这里可以随便添能标示就成.由于 nginx version 为字符所以 Type of information 选择为 text，其他选项一般默认即可.之后点击**Preprocessing**标签，由于此 item 数据来自 nginx status 这个 master,因此要使用预处理功能，对 nginx status 获取的 json 进行处理，得到我们需要的 nginxVersion 这个数据.
![prep](https://img.cactifans.com/wp-content/uploads/2017/12/D34514BF-DAE9-46B2-AF94-5B977FB11D4E.jpg)
如图 name 选择 JSON Path,Parameters 为需要获取的字段.具体使用方式可查看[item_value_preprocessing](https://www.zabbix.com/documentation/3.4/manual/config/items/item#item_value_preprocessing)说明，由于 nginxVersion 为第一层，Parameters 填写为$.nginxVersion即可，保存之后，过一会就可以看到nginxVersion数据已经采集了.
![status2](https://img.cactifans.com/wp-content/uploads/2017/12/9730C67B-F9CB-493B-8DF9-EBD033CE7609.jpg)
接下来再添加一个nginx连接数统计.点击create item，如图配置item
其他不变
![request1](https://img.cactifans.com/wp-content/uploads/2017/12/BA5B0E0D-5CB2-40CA-ACEC-C67EBD46E5F3.jpg)
由于nginix connections requests为一个数值类型，这里Type of information 选择Numeric类型.key可随便填写.**Preprocessing**标签如图配置
![rep](https://img.cactifans.com/wp-content/uploads/2017/12/A22EDC75-D1F7-4C2D-9BCA-4DA41012E061.jpg)
由于通过获取的json可以看到connections requests在json里为connections之内的requests，因此Parameters填写为$.connections.requests.从 nginx-module-vts 页面得知，这个 requests 为 The total number of requested client connections.即所有客户端连接，**为一个计数器**，这里我们需要统计为速率，因此这里点击 add，从上一步获取的 requests 数据再做一次计算，选择 Change per second 即之前的每秒变化（Req/s）即可.依照此方法可添加其他指标.最后效果如下：
最新数据
![last1](https://img.cactifans.com/wp-content/uploads/2017/12/113F81EE-6BE0-4200-AAC1-D12418D4E520.jpg)
请求汇总
![graph1](https://img.cactifans.com/wp-content/uploads/2017/12/A1F19141-7D55-4BE7-81AA-61040292C815.jpg)
（图上数据较大出为第一次没有添加 Change per second，造成统计所有请求数，后来查看 nginx-module-vts 文档得知，此数据为计数器，后续改动导致数据突降.由于是测试服务器基本无人访问数据为 0 属于正常）

## 拓展即思考

个人认为 Preprocessing 和 Dependent item 主要有以下几个好处和扩展之处:
**1.切割处理以前不好处理的数据**.之前通过 SNMP 获取到设备内存或者 cpu 使用率，获取数据往往为“8Mb”或“16%”等带单位的数据，这种类型不属于整型和浮点类型，因此只能作为字符处理，而作为字符就不能做成折线图，不能根据数值来做触发器.有了预处理功能就可以完美解决这个问题.
**2.item 数据可为 xml/json 等格式的数据，丰富了数据采集类型**.通过 Preprocessing 功能获取指定字段，存入数据.
**3.结合 3.4 的 Dependent items 可以提高监控效率.**通过 Dependent items 功能可一次返回多个 item 的数值，提高监控的效率，降低了系统负荷，减少网络流量
**4.多样化数据处理，可对采集数据做深层次处理.**可对一些暴露 http 接口的应用或者服务进行监控
**5.结合 UserParameter 功能，可进行分布式监控.**对网站/App 等进行分布式的监控.

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
