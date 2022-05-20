---
title: "zabbix微信告警"
date: 2016-01-27 11:15:57
tags:
  - zabbix
  - 微信
categories: [zabbix]
---

前面写了一个 zabbix 微信告警的，用的我的企业号，后来发现用的人太多消息都超过限制了，应大家要求发布个可以用主机企业号的发送程序，填自己的企业号就可发送微信告警消息！使用 go 语言开发（感谢[老司机](https://github.com/chanxuehong/wechat)提供的微信 sdk)

## 首先你得有个企业号！！！！

关于企业号的申请，什么是 corpid，secret，agentid，微信号，用户账号等等问题我就不科普了，大家可以上腾讯的[企业号开发者中心](http://qydev.weixin.qq.com/wiki/index.php?title=%E9%A6%96%E9%A1%B5)查看，或者查看 itnihao 的一篇 blog，[http://itnihao.blog.51cto.com/1741976/1733245](http://itnihao.blog.51cto.com/1741976/1733245)图文并貌写的很清楚。

## 下载程序

下载地址：
[zabbix_weixin.x86.tar.gz](https://dl.cactifans.com/tools/zabbix_weixin.x86.tar.gz)（Linux32 位版本）
[zabbix_weixin.x86_64.tar.gz](https://dl.cactifans.com/tools/zabbix_weixin.x86_64.tar.gz)（Linux64 位版本）

## 部署步骤

**下载程序到你的 zabbix server 的 AlertScriptsPath 目录下**。不知道什么是 AlertScriptsPath 目录，不知道怎么配置的，直接看官方文档！！！[zabbix server 配置文件](https://www.zabbix.com/documentation/2.4/manual/appendix/config/zabbix_server)
如果之前没有设置过 AlertScriptsPath，设置之后要重启 zabbix server
假设我的 zabbix server 的 AlertScriptsPath 目录为/usr/local/zabbix/alertscripts

```bash
wget https://dl.cactifans.com/tools/zabbix_weixin.x86_64.tar.gz
tar zxvf zabbix_weixin.x86_64.tar.gz
mv zabbix_weixin/weixin .
chmod a+x weixin
mv zabbix_weixin/weixincfg.json /etc/
rm -rf zxvf zabbix_weixin.x86_64.tar.gz
rm -rf zabbix_weixin/
```

接下来一步很重要，编辑/etc/weixincfg.json 文件，配置你的企业号 corpid，secret，agentid，

```
{
"corp": {
        "corpid": "wxxxxxx",
        "secret": "Vn6dxxxx",
        "agentid": 1
    }
}
```

**不知道哪里看 corpid，scret，agentid 的直接看 itnihao 的文章，不要再问我！**
AgentId
![AgentId](https://img.cactifans.com/wp-content/uploads/2016/01/55102EE3-8C8C-4B65-ABA1-C80FCC980F48.jpg)

## 测试

```
/usr/local/zabbix/alertscripts/weixin xxx subject body
```

解释一下（**这里我只是演示，具体的你要替换成你自己的信息，切不可按图索骥**）

```
xxx为你的微信账号！注意不是微信号！也不是微信昵称！当然你也可以把用户账号设置成微信号或者微信昵称，自己设置！
subject 告警主题
boyd 告警闲情
```

介于多数人分不清楚，这里解释一下：
**在微信企业号里，成员要关注企业号，需要审核，审核之后每个人会赋予一个账号。**
个人账号
![个人账号](https://img.cactifans.com/wp-content/uploads/2016/01/26BA328B-5D66-4727-B5F5-D2B0AA61B897.jpg)
如果发送显示“OK”，表示发送成功，应该就会收到消息！

## zabbix 设置

先添加微信到告警媒介
![告警媒介](https://img.cactifans.com/wp-content/uploads/2016/01/62CCF386-0FE1-4AE7-833C-D7F88405B539.jpg)
3.0 需要额外配置下，不配置不能发送!!!
![用户](https://img.cactifans.com/wp-content/uploads/2016/01/D92768163CB1.jpg)
关联到用户
![用户](https://img.cactifans.com/wp-content/uploads/2016/01/27FED34F-E27A-485E-90E5-36DFAEDA95C8.jpg)
告警内容定制
![内容](https://img.cactifans.com/wp-content/uploads/2016/01/64E42502-131B-4DE0-B81A-4716EAAB92DF.jpg)
注意：收件人哪里填需要收消息的人的个人账号，多个人中间用“｜”号隔开，如图所示
告警内容是我自己定制的,大家可以参考我的，直接复制过去用

```
告警主题：
[{TRIGGER.SEVERITY}]服务器:{HOSTNAME1}发生:{TRIGGER.NAME}故障！
告警内容：
告警主机: {HOSTNAME1}
主机分组: {TRIGGER.HOSTGROUP.NAME}
告警时间: {EVENT.DATE} {EVENT.TIME}
告警等级: {TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目: {TRIGGER.KEY1}
问题详情: {ITEM.NAME}:{ITEM.VALUE}
当前状态: {TRIGGER.STATUS}
事件ID: {EVENT.ID}
```

告警恢复内容

```
恢复主题：
[{TRIGGER.SEVERITY}]服务器:{HOSTNAME1}{TRIGGER.NAME}已恢复！
恢复内容：
告警主机: {HOSTNAME1}
主机分组: {TRIGGER.HOSTGROUP.NAME}
告警时间: {EVENT.DATE} {EVENT.TIME}
告警等级: {TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目: {TRIGGER.KEY1}
问题详情: {ITEM.NAME}:{ITEM.VALUE}
当前状态: {TRIGGER.STATUS}
事件ID: {EVENT.ID}
```

设置好之后，设置动作时，掉用 weiixn 就是了
![调用](https://img.cactifans.com/wp-content/uploads/2016/01/4998E246-A2D0-4136-AC89-75316ECA7B60.jpg)
至此设置完成！

## 最终效果

![效果](https://img.cactifans.com/wp-content/uploads/2016/01/%E6%B6%82%E9%B8%A6_Screenshot_2016-01-27-12-03-45-359_%E5%BE%AE%E4%BF%A1.png)

FAQ：
A.测试不能通过，返回 errcode！

1.检查/etc/weixincfg.json 文件里的 corpid，secert，agentid 配置是否正确 2.检查接受者企业账号是否正确 3.检查接受着是否在这个应用的通讯录里

B.zabbix 不能收到告警消息 1.检查发送程序有无可执行权限 2.检查发送程序是否在 zabbix server 的 AlertScriptsPath 目录下 3.检查是否关联到用户 4.检查是否掉用了发送动作

C. 发送限制 1.发送频率基本可以满足需求,没有别的限制。 2.每日发送次数有一定限制，具体与企业号关注人数有关，详情查看企业号开发文档

zabbix 二次开发和技术支持事宜请联系本人
mail:lovecanghai#gmail.com

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
