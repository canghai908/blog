---
title: Zabbix在Docker中的应用和监控
date: 2018-12-28 17:38:00
tags:
  - zabbix
  - docker
categories: [zabbix]
---

最近以来很多人在群里问，zabbix 能不能跑在 Docker 里？如何使用 zabbix 来监控 Docker 等一系列问题。回答是肯定的：能！本次为大家介绍如何使用，同时本内容也是本人在 Zabbix Conference China 2018 WorkShop 里的内容。

## 一.如何使 Zabbix 跑在 Docker 里

Zabbix 官方很早之前就提供里 Zabbix 的 Docker 镜像，而且提供里具体的配置及文件。具体地址：https://github.com/zabbix/zabbix-docker 官方提供三种 Docker 基础镜像的版本，分别为：

- alpine
- centos
- ubuntu

基础镜像在使用上没有太大区别，这里推荐大家使用 alpine，这是一个简化的 linux 版本，最小体积只有 30MB 多，建议大家使用。官方提供提供了 docker-compose 的编排文件，可以使用 docker-compose 编排工具，"一键"启动一套 Zabbix 系统。其中包括以下组件：

- zabbix-server
- zabbix-agent
- zabbix-proxy
- zabbix-web
- zabbix-java-gateway
- zabbix-snmptraps

### 1.Docker 基础环境配置

环境
OS：CentOS 7 x86_64
Docker: docker-ce-18.09.0
Step1 安装必要的一些系统工具

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

Step2 添加软件源信息

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

Step3 更新并安装 Docker-CE

```bash
yum makecache fast
yum -y install docker-ce
```

Step4 开启 Docker 服务

```bash
systemctl start docker
```

Step5 设置开机启动

```bash
systemctl enable docker
```

Step6 配置 docker 镜像加速

```bash
vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://72idtxd8.mirror.aliyuncs.com"]
}
```

Step 7 重启 docker

```bash
systemctl restart docker
```

### 2.Docker-compose 安装配置

环境
OS：CentOS 7 x86_64
Docker: docker-ce-18.09.0
Docker-compose：docker-compose version 1.23.1
docker-compose 是 docker 推出的一款编排工具，由于 zabbix 有很多组件，zabbix server，zabbix-web，zabbix-proxy,db 等组件，通过 docker-compose 的配置文件，可以统一编排，做到一键启动一套 Zabbix 组件。
Step1 安装 Docker-compose

```bash
curl "https://dl.cactifans.com/zabbix_docker/docker-compose" -o /usr/bin/docker-compose
chmod a+x /usr/bin/docker-compose
```

Step2 查看 docker-compose 版本

```bash
docker-compose version
```

Step3 安装 git 等工具

```bash
yum install git wget telnet net-tools -y
```

Step4 下载 zabbix docker 仓库文件并切换到 4.0 分支

```bash
cd /opt
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker/
git checkout 4.0
```

由于 github 访问较慢，我已 git 一份到我服务器，大家可以使用以下命令下载使用，只是下载地址变化，其他么有大的变化。

```bash
cd /opt
wget https://dl.cactifans.com/zabbix_docker/zabbix-docker.tar.gz
tar zxvf zabbix-docker.tar.gz
cd zabbix-docker/
git checkout 4.0
```

### 3.启动 zabbix server

zabbix 官方提供的 docker-compose 文件有很多，导致大家感到困惑，下面为大家解释一下：

```bash
docker-compose_v3_alpine_mysql_latest.yaml
```

- v3 为 docker-compose 版本，分 v3，v2，与 docker-compose 版本和 docker 版本有关系，具体对应关系在这里查看https://docs.docker.com/compose/compose-file/compose-versioning/
- alpine 为基础镜像类型，三种类型 alpine/centos/ubuntu 可选，三种镜像在 Zabbix 使用上没有任何区别，区别的的只有镜像大小及操作系统区别，推荐使用 alpine
- myql 为 zabbix server 所使用的数据库类型，目前有 MySQL/PostgreSQL 二种，推荐使用 mysql
- latest 表示为使用官方的最新镜像，local 是下载本地进行 build 镜像，如网络不好，建议不要尝试,时间较长，很容易失败

本次使用 docker-compose_v3_alpine_mysql_latest.yaml 配置文件启动 Zabbix
Step1 启动 zabbix server 组件

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml up -d
```

Step2 查看运行状态

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml ps
```

启动之后即可使用http://ip 直接访问 zabbix server，默认账号密码为

```bash
账号：Admin
密码：zabbix
```

### 4.基本配置

安装好之后，部分配置可根据实际需求修改

#### 1.修改 web 端口

Step1 修改 docker-compose_v3_alpine_mysql_latest.yaml 文件

```bash
vi docker-compose_v3_alpine_mysql_latest.yaml
```

修改端口为 8812

```bash
zabbix-web-apache-mysql:
  image: zabbix/zabbix-web-apache-mysql:alpine-4.0-latest
  ports:
   - “80:80"        //修改为8812:80
   - “443:443”
```

Step2 停止 zabbix server

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml down
```

Step3 启动 zabbix server

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml up -d
```

即可使用http://ip:8812 访问应用

#### 2.修改时区

Step1 修改.env_web

```bash
vi .env_web
```

修改为 Asia/Shanghai

```bash
PHP_TZ=Europe/Riga
修改为 PHP_TZ=Asia/Shanghai
```

Step2 停止 zabbix server

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml down
```

Step3 启动 zabbix server

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml up -d
```

#### 3.修改字体为中文

由于默认镜像中文字体乱码，需要添加中文字体，重新 build 镜像，此过程较长，需要网络良好。
Step1 修改配置文件

```bash
vi zabbix-docker/web-apache-mysql/alpine/Dockerfile
```

修改成如下内容

```bash
ADD conf/etc/zabbix/web/msty.ttf /usr/share/fonts/ttf-dejavu/msty.ttf
ln -s /usr/share/fonts/ttf-dejavu/msty.ttf /usr/share/zabbix/fonts/graphfont.ttf
```

Step2 下载字体文件到 conf/etc/zabbix/web/目录

```bash
wget https://dl.cactifans.com/zabbix_docker/msty.ttf
```

Step3 生成镜像

```
zabbix-docker/web-apache-mysql/alpine/build.sh
```

step4 修改 docker-compose 文件，并启动

```bash
vi  docker-compose_v3_alpine_mysql_latest.yaml
```

镜像地址修改为刚才 build 的

```bash
image: zabbix-web-apache-mysql:alpine-latest
```

step5 启动 zabbix server

```bash
docker-compose -f docker-compose_v3_alpine_mysql_latest.yaml up -d
```

## 二.使用 Zabbix 监控 Docker

随着 Docker 的流行，监控 Docker 已势在必行，使用 Zabbix 可以利用 LLD（自动发现）自动监控宿主机上所运行的所有 Docker 状态。具体地址可查看https://github.com/monitoringartist/zabbix-docker-monitoring/
基本原理如下
![1](https://img.cactifans.com/wp-content/uploads/2018/12/dockbix-agent-xxl-schema.png)

### 1.部署方式

提供了二种部署方式 1.使用已有 Zabbix Agent 加载 Docker 监控模块方式.如已在 Docker 宿主机上安装 Agent，可直接修改配置文件，加载对应的 Docker 监控模块，重新启动 Agent 即可。 2.使用加载了 Zabbix Agent 的 Docker 镜像方式.如未安装 Zabbix Agent，可直接使用包含了 Zabbix Agent 的镜像即可。

### 2.模块方式

Zabbix 模块插件是很好用的，可使用 C 语言编写一个模块，直接加载模块即可使用，个人认为使用模块有以下好处:

- 避免手动添加 UserParameter 带来的繁琐。模块自动添加 UserParameter，无需手动添加
- 避免脚本泄漏，保存重要信息。如部分脚本里保存数据库账号密码，普通用户可直接查看内容，加密之后，脚本执行又需要解密，比较繁琐，使用模块可避免此类问题。
- 统一管理自定义监控项。监控某类指标，只需加载对应的模块即可，按需加载

在 Zabbix Agent 里添加模块只需修改以下二项即可

```bash
LoadModulePath=/usr/local/modules
LoadModule=docker.so
```

第一项为模块路径，第二项为模块文件。修改之后重启 Zabbix Agent 即可。
监控 Docker 模块可在https://github.com/monitoringartist/zabbix-docker-monitoring 下载，注意对应的版本。**网站提供的模块有些有错误，需要自行编译。**

### 3.使用 Docker Agent 方式

使用以下命令即可启动一个 Agent，即可监控宿主机器上所有运行的 Docker 容器

```bash
docker run \
--name=dockbix-agent-xxl \
--net=host \
--privileged \
-v /:/rootfs \
-v /var/run:/var/run \
--restart unless-stopped \
-e "ZA_Server=<ZABBIX SERVER IP/DNS NAME/IP_RANGE>" \
-e "ZA_ServerActive=<ZABBIX SERVER IP/DNS NAME>" \
-e "ZA_StartAgents=10" \
-e "ZA_Timeout=30" \
-d monitoringartist/dockbix-agent-xxl-limited:latest
```

- ZA_Server 修改为你的 Zabbix ServerIP
- ZA_ServerActive 修改为你的 Zabbix ServerIP

即可完成 Agent 部署

### 4.关联模版

在 Zabbix 主机上导入模版，并关联主机。模版下载地址：
https://dl.cactifans.com/zabbix_docker/Zabbix-Template-App-Docker.tar.gz ，下载之后解压导入模版，添加主机即可。主要使用 2 个模版，一个为主动,一个为被动

- Zabbix-Template-App-Docker-active.xml 主动模版
- Zabbix-Template-App-Docker.xml 被动模式

添加主机的主机名为**宿主机名称**，也可以通过 docker 日志查看。

### 5.效果

CPU
![1](https://img.cactifans.com/wp-content/uploads/2018/12/1545989160664.jpg)
Cache Memory
![2](https://img.cactifans.com/wp-content/uploads/2018/12/1545989351987.jpg)
RSS Memory
![3](https://img.cactifans.com/wp-content/uploads/2018/12/1545989329557.jpg)
Container Status
![4](https://img.cactifans.com/wp-content/uploads/2018/12/1545989196499.jpg)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
