---
title: Open-Falcon微信告警组件
date: 2018-07-19 16:00:25
tags: [open-falcon, 微信]
categories: [open-falcon]
---

## Open-Falcon 微信告警组件

在此
感谢 @laiwei 的https://github.com/open-falcon/mail-provider 项目代码支持
感谢 @chanxuehong 老司机的微信 SDK 支持

## 申请微信企业号

在https://work.weixin.qq.com/ 注册微信企业号，不需要认证即可

### 配置企业号

在企业号里建立应用,依此点击：企业应用-创建应用。可见范围里添加成员
获取企业 ID：点击我的企业--获取企业 ID

![1](https://img.cactifans.com/wp-content/uploads/2018/07/2.jpg)
获取应用 AgentId 及应用 Secret
![2](https://img.cactifans.com/wp-content/uploads/2018/07/1.jpg)

## 部署 Falcon-wechat

Falcon-wechat 可部署在 Falcon-Alarm 机器，也可部署在独立机器，使用以下命令部署

```bash
 mkdir -p /usr/local/falcon-wechat
 cd /usr/local/falcon-wechat
 wget https://dl.cactifans.com/open-falcon/falcon-wechat-0.0.1.tar.gz
 tar zxvf falcon-wechat-0.0.1.tar.gz
```

修改配置文件 cfg.json 配置文件,将 corpid 修改为你的企业 ID，secret 修改为你应用的 secret，agentid 修改为你的 AgentId，并保存

```json
{
  "debug": true,
  "http": {
    "listen": "0.0.0.0:9527",
    "token": ""
  },
  "wechat": {
    "corpid": "wxa7c63522727b6bf0",
    "secret": "d5S-_XGVd-5HA0o9bkvMMK5Wh1qwCC-YQei4WcL9hSM",
    "agentid": 1
  }
}
```

启动 falcon-wechat

```bash
./control start
```

启动信息

> falcon-wechat started..., pid=13875

查看日志

```
./control tail
```

如看到以下信息表示启动成功

> 2018/07/19 14:44:28 config.go:64: load configuration file cfg.json successfully
> 2018/07/19 14:44:28 http.go:25: http listening 0.0.0.0:9527

## 配置 Open-Falcon

### 配置 Alarm 组件

修改 Open-Falcon 的 Alarm 组件 config 目录下的配置文件 cfg.json，将 IM 段修改为以下内容

```json
	"im": "http://127.0.0.1:9527/wechat",
```

> 如果你修改了 falcon-wechat 的默认端口，请注意修改。如 falcon-wechat 和 Alarm 组件为不同机器，注意修改 IP 地址。
> 修改之后重启 Alarm 服务，使其生效

### 配置 IM 信息

在 Dashboard 里，为用户配置 IM 账号为户**用户账号**！**用户账号**！
**用户账号** 不是微信号，重要事情说三遍！
![3](https://img.cactifans.com/wp-content/uploads/2018/07/3.jpg)
用户账号
![4](https://img.cactifans.com/wp-content/uploads/2018/07/4.jpg)

## 效果

告警
![5](https://img.cactifans.com/wp-content/uploads/2018/07/5.jpg)
微信
![6](https://img.cactifans.com/wp-content/uploads/2018/07/6.png)

## 注意事项

1.由于使用企业微信发送消息接口实现，接口调用速率有限制。注意控制消息发送频率，目前每次发消息都会请求一次 Access_token，后续优化。 2.由于未认证企业发送消息数量与人员有关，建议控制频率。具体查看官网 API 文档https://work.weixin.qq.com/api/doc#10785
