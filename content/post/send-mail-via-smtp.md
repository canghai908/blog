---
title: zabbix邮件告警
date: 2016-01-21 17:06:18
tags:
  - zabbix
  - smtp
  - mail
categories: [zabbix]
---

使用 zabbix 邮件发送告警消息，老是遇到发送程序出现问题，因此使用 go 结合开源的邮件库，写了一个 smtp 发邮件的程序（CentOS 6.4 X64 位测试通过）
下载地址：[zabbix_mail.x86_64.tar.gz](https://dl.cactifans.com/tools/zabbix_mail.x86_64.tar.gz)
使用方法：
zabbix alertscripts 脚本路径为/usr/local/zabbix/alertscripts

```
cd /usr/local/zabbix/alertscripts
wget https://dl.cactifans.com/tools/zabbix_mail.x86_64.tar.gz
tar zxvf zabbix_mail.x86_64.tar.gz
rm -r zabbix_mail.x86_64.tar.gz
mv zabbix_mail/mail .
chmod a+x mail
mv zabbix_mail/cfg.json /etc/
```

编辑/etc/cfg.json 配置 SMTP 邮件服务器信息

```
{
"smtp": {
        "username": "alarm@126.com",
        "password": "password",
        "description": "运维监控",
        "host": "smtp.126.com",
        "port": 587
    }
}

```

根据实际情况填写，最好使用企业内部邮件服务器的 smtp，163，126 发送邮件过多会屏蔽，请慎重使用！！！
测试
执行

```
/usr/local/zabbix/alertscripts/mail xxxx@126.com 邮件主题 邮件内容
```

xxx@126.com 为接受人邮件地址,后面接邮件主题，邮件内容。如果能收到邮件表示发送成功!
mail 后面可跟三个参数，与 zabbix 一致，可查看 zabbix 官方文档
[Zabbix Custom alertscripts](https://www.zabbix.com/documentation/2.4/manual/config/notifications/media/script)
参数解释

```
 $1 邮件接收人，过个接收人用；号分割
 $2 邮件主题
 $3 邮件内容
```

在 zabbix 里添加告警脚本
添加告警媒介
![ 添加告警媒介](https://img.cactifans.com/wp-content/uploads/2016/01/F29BC2EA-E01F-41FE-AF61-22240D77DB1A.jpg)
关联到用户
![关联到用户](https://img.cactifans.com/wp-content/uploads/2016/01/5207E7CC-44CB-44F2-9145-D2B994E6956C.jpg)
关联到动作
![关联到动作](https://img.cactifans.com/wp-content/uploads/2016/01/33801C4B-682A-4A91-9A62-DCF39DC95555.jpg)
配置完成！

具体设置也可参考我的另外一篇文章
[zabbix 通过微信告警](http://canghai.coding.io/2015/04/18/Configure-Zabbix-to-send-notifications-throught-weixin/)
