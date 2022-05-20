---
title: "Zabbix5.0版本Agent2安装"
date: 2020-05-19T16:59:23+08:00
tags: [zabbix, agent2]
categories: [zabbix]
---

## Zabbix Agent2 介绍

Zabbix 5.0 版本推出了使用 go 语言重写的 Agent2，也是 5.0 版本新特性，Agent2 有如下特性：

- 完成的插件框架支持，可扩张服务及应用监控
- 支持灵活的采集周期调度
- 更高效的数据采集及传输
- 可完全替换先有的 agent
- .....

特性较多，建议使用。由于使用 go 语言编写,编译安装与之前版本有所区别。Agent2 默认使用的 10050 端口，与 Zabbix Agent 端口一样，不修改端口情况下，同一台机器不能同时启动 Zabbix Agent 与 Zabbix Agent2。

### 安装

安装可使用 yum 和编译安装，对于新手，建议使用 yum 安装。

### yum 安装

参照上篇[Zabbix 5.0 LTS 版本安装](https://blog.cactifans.com/2020/05/17/Zabbix-5.0-LTS-%E7%89%88%E6%9C%AC%E5%AE%89%E8%A3%85/) 配置好 yum 源，使用以下命令即可安装 Zabbix Agent2

```
yum install zabbix-agent2 -y
```

默认配置文件为

```
/etc/zabbix/zabbix_agent2.conf
```

默认二进制文件为

```
/usr/sbin/zabbix_agent2
```

使用以下命令启动 Agent2 并配置开机启动

```
systemctl enable --now zabbix-agent2
```

### 编译安装

安装 gcc 等基础编译环境，由于使用 go 编写，因此需要配置 go 编译环境，下载并配置 go 语言编译环境

```
cd /opt
wget https://dl.google.com/go/go1.14.3.linux-amd64.tar.gz
tar zxvf go1.14.3.linux-amd64.tar.gz -C /usr/local/
echo "export PATH=$PATH:/usr/local/go/bin" >>/etc/profile
source /etc/profile
go env
```

最后显示如下，表明 go 语言环境配置成功。

```
GO111MODULE=""
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOENV="/root/.config/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOINSECURE=""
GONOPROXY=""
GONOSUMDB=""
GOOS="linux"
GOPATH="/root/go"
GOPRIVATE=""
GOPROXY="https://proxy.golang.org,direct"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GCCGO="gccgo"
AR="ar"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD=""
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build821720893=/tmp/go-build -gno-record-gcc-switches"
```

开启 go mod，由于编译过程需要联网下载依赖包，配置 go mod 代理

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

下载 zabbix 5.0 源码

```
cd /opt
wget https://cdn.zabbix.com/zabbix/sources/stable/5.0/zabbix-5.0.0.tar.gz
tar zxvf zabbix-5.0.0.tar.gz
cd zabbix-5.0.0
```

如果只是要编译 agent2，直接加-enable-agent2 参数即可

```
./configure --prefix=/usr/local/zabbix -enable-agent2
make
make install
```

编译过程中有错误一定要关注，其中需要联网下载依赖包，耐心等待安装完成。
默认配置文件

```
/usr/local/zabbix/etc/zabbix_agent2.conf
```

二进制程序

```
/usr/local/zabbix/sbin/zabbix_agent2
```

配置 systemd 启动文件

```
vi /lib/systemd/system/zabbix-agent2.service
```

内容如下

```
[Unit]
Description=Zabbix Agent 2
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/usr/local/zabbix/etc/zabbix_agent2.conf"
EnvironmentFile=-/etc/sysconfig/zabbix-agent2
Type=simple
Restart=on-failure
PIDFile=/tmp/zabbix_agent2.pid
KillMode=control-group
ExecStart=/usr/local/zabbix/sbin/zabbix_agent2 -c $CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
User=zabbix
Group=zabbix

[Install]
WantedBy=multi-user.target
```

配置启动并设置开机启动

```
systemctl enable --now zabbix-agent2
```

## 配置

zabbix agent2 的配置与之前的 zabbix agent 配置基本一致

```
Server=172.16.66.11
ServerActive=172.16.66.11
Hostname=node16
```

Server 和 ServerActive 配置为 zabibx server 或 zabbix proxy 地址，Hostname 配置为主机名即可。
Agent2 没有组件依赖，可直接拷贝编译好的二进制文件和配置文件在其他主机上运行即可。

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
