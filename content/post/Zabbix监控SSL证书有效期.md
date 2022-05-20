---
title: Zabbix监控SSL证书有效期
date: 2018-05-16 14:27:30
tags: [SSL证书, Zabbix]
categories: [zabbix]
---

## 设计初衷

由于业务需要，最近通过 Let's Encrypt 申请了一些 SSL 证书，而证书有效期为 3 个月，需要在证书到期之前 renew。由于域名较多经常忘记 renew，导致证书过期，因此想通过 Zabbix 的方式监控证书的到期时间，提前告警以便即时 renew 证书

## 使用说明

脚本下载地址；
Linux kernel 3.x x86_64: https://dl.cactifans.com/zabbix/zabbix_sslooker.kernel_3.10.0.x86_64.tar.gz
Linux kernel 2.x x86_64:https://dl.cactifans.com/zabbix/zabbix_sslooker.kernel_2.6.32.x86_64.tar.gz
Windows AMD64:https://dl.cactifans.com/zabbix/zabbix_sslooker.windows-amd64.zip

注意事项： 1.获取证书有效期为**小时** 2.**自签发证书**暂不支持检测

### Zabbix Agent 配置

下载对应的脚本到安装了 Zabbix Agent 并可以访问到检测证书网站的机器

```bash
cd /usr/local/src/
wget https://dl.cactifans.com/zabbix/zabbix_sslooker.kernel_3.10.0.x86_64.tar.gz
tar zxvf zabbix_sslooker.kernel_3.10.0.x86_64.tar.gz
chmod a+x sslooker
```

修改 agent 的配置文件 zabbix_agent，增加如下内容

```bash
UserParameter=sslcheck[*],/usr/local/src/sslooker $1 $2
```

sslcheck 为 zabbix 的 item key，/usr/local/zabbix/share/sslooker 为下载解压后的脚本可执行程序
添加之后，重启 Zabbix Agent
在 Zabbix Server 上通过 Zabbix get 测试是否正常

```bash
zabbix_get -s  127.0.0.1 -k sslcheck[baidu.com,443]
```

如图
![1](https://img.cactifans.com/wp-content/uploads/2018/05/sslooker1.jpg)
由于我使用 zabbix server 来检测证书，所以直接 get 本机的 agent 地址
sslcheck 为刚才脚本里定义的 key，方括号内为参数,第一个为域名，第二个为端口。返回数值为证书的**有效时间**，单位为**小时**

### Zabbix Server 配置

进入 zabbix server，在对应的机器上建立对应的 Item 及 Trigger 即可告警。这里以检测 baidu 网站证书为例子，并设置过期 48 小时之前告警。
设置 Item
![item](https://img.cactifans.com/wp-content/uploads/2018/05/sslooker2.jpg)
设置 Trigger
![trigger](https://img.cactifans.com/wp-content/uploads/2018/05/sslooker3.jpg)
最新数据
![last](https://img.cactifans.com/wp-content/uploads/2018/05/sslooker4.jpg)
多个域名可以通过建立多个 Item 的方式监控，或通过主机宏的方式监控

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
