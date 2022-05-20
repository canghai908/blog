---
title: Nightingale开源监控系统2-客户端部署
tags: [nightingale]
categories: [nightingale]
date: 2020-11-10 10:27:30
---

上期介绍了夜莺监控服务端的部署安装，本次主要介绍 Linux 监控客户端的独立安装及配置。

## 客户端部署

打包编译好的监控客户端程序，部署到需要监控的操作系统。客户端程序目录结构如下

```
.
├── etc
│   ├── address.yml
│   ├── agent.yml
│   ├── identity.yml
│   ├── agent.service
└── n9e-agent
```

etc 目录下为 agent 所需要的 3 个 yml 格式配置文件，agent.service 为 agent 启动的 systemd 脚本文件，建议使用 systemd 方式启动 agent。n9e-agent 为二进制程序。  
将以上程序放入/usr/local/n9e 目录下，agent.yml 默认不用修改，identity.yml 文件里如果 shell 部分执行出错或者较慢，建议配置 specify 为机器 ip 即可。

```
# 用来做心跳，给服务端上报本机ip
ip:
  specify: "172.16.66.22"
  shell: ifconfig `route|grep '^default'|awk '{print $NF}'`|grep inet|awk '{print $2}'|head -n 1

# MON、JOB的客户端拿来做本机标识
ident:
  specify: "172.16.66.22"
  shell: ifconfig `route|grep '^default'|awk '{print $NF}'`|grep inet|awk '{print $2}'|head -n 1
```

由于 agent 需要与 n9e 后端各个服务进行通行，因此需要修改 address.yml 文件,把文件中 127.0.0.1 的地址替换为 n9e 服务端 ip 即可

```
---
rdb:
  http: 0.0.0.0:8000
  addresses:
    - 172.16.66.16

ams:
  http: 0.0.0.0:8002
  addresses:
    - 172.16.66.16

job:
  http: 0.0.0.0:8004
  rpc: 0.0.0.0:8005
  addresses:
    - 172.16.66.16

monapi:
  http: 0.0.0.0:8006
  addresses:
    - 172.16.66.16

transfer:
  http: 0.0.0.0:8008
  rpc: 0.0.0.0:8009
  addresses:
    - 172.16.66.16

tsdb:
  http: 0.0.0.0:8010
  rpc: 0.0.0.0:8011

index:
  http: 0.0.0.0:8012
  rpc: 0.0.0.0:8013
  addresses:
    - 172.16.66.16

judge:
  http: 0.0.0.0:8014
  rpc: 0.0.0.0:8015
  addresses:
    - 172.16.66.16

agent:
  http: 0.0.0.0:2080
```

修改 agent.service 文件里的 n9e-agent 安装目录

```
[Unit]
Description=n9e agent
After=network-online.target
Wants=network-online.target

[Service]
# modify when deploy in prod env
User=root
Group=root

Type=simple
Environment="GIN_MODE=release"
ExecStart=/usr/local/n9e/n9e-agent
WorkingDirectory=/usr/local/n9e

Restart=always
RestartSec=1
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

启动 n9e-agent 服务并配置开机自启动

```
cp agent.service /lib/systemd/system/n9e-agent.service
systemctl daemon-reload
systemctl enable --now n9e-agent
```

至此监控客户端安装完成。可看查看 logs 目录下的日志文件，如出现错误可根据日志排查配置是否正确

## 页面配置

n9e-agent 启动后，登录 web 页面，点击资产管理中心-设备管理，即可看到所监控设备已经在线
![1](https://img.cactifans.com/wp-content/uploads/2020/11/1605019897292.jpg)
选择设备，点击批量操作——修改租户操作，修改为内置租户，即可把设备划分到特定的租户。
![2](https://img.cactifans.com/wp-content/uploads/2020/11/1605015325842.jpg)
点击用户资源中心——组织资源树并配建立资源树，选中建立好的树节点，选择资源管理——批量操作——挂载资源——输入监控设备的 IP 即可完成资源的挂载。
![3](https://img.cactifans.com/wp-content/uploads/2020/11/1605015650688.jpg)
后续告警策略及权限分配都根据资源树进行。可点击树资源查看对应设备及监控数据。
![4](https://img.cactifans.com/wp-content/uploads/2020/11/1605020481867.jpg)
至此完成设备的添加配置。
下期介绍端口及进程监控告警配置
