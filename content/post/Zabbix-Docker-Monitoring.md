---
title: Zabbix 监控 docker
date: 2017-12-10 21:16:33
tags: [zabbix, docker, 容器监控]
categories: [zabbix]
---

以前使用 cadvisor 监控 Docker 容器状态,最近看到可以使用 Zabbix Module 的方式，通过部署一个 zabbix agent 的 docker 容器来监控宿主机器和宿主机器上 docker 的状态。原文可在 https://github.com/monitoringartist/zabbix-docker-monitoring 处查看，我只是搬运工!

### 1.使用 Zabbix Agent Docker 进行监控

在需要监控的宿主机器上运行运行 Agent 容器

```bash
docker run \
  --name=dockbix-agent-xxl \
  --net=host \
  --privileged \
  -v /:/rootfs \
  -v /var/run:/var/run \
  --restart unless-stopped \
  -e "ZA_Server=192.168.0.252" \
  -e "ZA_ServerActive=192.168.0.252" \
  -d hub.c.163.com/canghai809/dockbix-agent-xxl-limited:latest
```

192.168.0.252 为已经安装好的 zabbix server 的 ip 地址。具体根据自身情况修改，最好填写 IP 地址。由于 Docker 官方的 hub 在国内下载较慢，我已同步到网易蜂巢镜像，提供多个版本主要为 Zabbix Agent 版本区别和 Docker API 版本区别。
Agent 不同版本镜像

```bash
docker pull hub.c.163.com/canghai809/dockbix-agent-xxl-limited:latest
docker pull hub.c.163.com/canghai809/dockbix-agent-xxl-limited:3.2-2
docker pull hub.c.163.com/canghai809/dockbix-agent-xxl-limited:3.2-1
```

运行之后，使用

```
docker logs -f xxxxx
```

xxxx 为刚运行容器的 id。查看日志，确认运行正常。

### 2.使用 Zabbix 模块方式进行监控

如果不想使用 Agent 的 Dcoker 镜像来监控，可以直接在 Agent 上通过加载 Zabbix Module 的方式监控，添加模版即可。
Zabbix Docker module 下载

|    OS    |                                     Module for Zabbix 3.4                                     |                                     Module for Zabbix 3.2                                     |                                     Module for Zabbix 3.0                                     |
| :------: | :-------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------------: |
| CentOS 7 | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos7/3.4/zabbix_module_docker.so) | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos7/3.2/zabbix_module_docker.so) | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos7/3.0/zabbix_module_docker.so) |
| CentOS 6 | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos6/3.4/zabbix_module_docker.so) | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos6/3.2/zabbix_module_docker.so) | [Download](https://dl.cactifans.com/zabbix/zabbix_module/centos6/3.0/zabbix_module_docker.so) |

下载对应的版本之后，在 Agent 配置文件里添加载模块

```bash
LoadModulePath=/usr/local/lib/zabbix/agent/
LoadModule=zabbix_module_docker.so
```

注意模块路径，按照自己的实际路径修改。并重启 agent,之后在 zabbix 里添加主机，关联 docker 模版即可.

### 3.Zabbix Server 配置

在 zabbix server 上导入监控 docker 的模版，一共 2 个模版,下载后解压
模版下载地址:
https://dl.cactifans.com/zabbix/Zabbix-Template-App-Docker.tar.gz
我使用主动模式，因此导入 Zabbix-Template-App-Docker-active.xml 这个模版

| 模板名称                              | 备注         |
| :------------------------------------ | :----------- |
| Zabbix-Template-App-Docker-active.xml | 主动模式模版 |
| Zabbix-Template-App-Docker.xml        | 被动模式模版 |

在 zabbix server 里添加主机
![添加主机](https://img.cactifans.com/wp-content/uploads/2017/12/035A26BD-2F06-49B7-948B-CE6FE839CA3D.jpg)
这里的机器名为使用 hostname 命令查到的机器名。关联 Linux OS 模版和 Zabbix-Template-App-Docker-active
![关联主机](https://img.cactifans.com/wp-content/uploads/2017/12/AFCEB933-09F0-4BE4-B0AF-3720D2A96772.jpg)
大概 10 分钟左右就可以看到监控效果
![效果图1](https://img.cactifans.com/wp-content/uploads/2017/12/FC1EFAFD-D375-4B6D-85EC-C42215B603E7.jpg)
![效果图2](https://img.cactifans.com/wp-content/uploads/2017/12/91C130BE-68E5-4203-91AE-C57AB6065FC8.jpg)
目前监控指标比较基本。提供了其余的监控指标，可以在https://github.com/monitoringartist/zabbix-docker-monitoring/blob/master/README.md处查看具体使用方法。

### 4.Debug

如出现无法监控的现象,有可能是 Docker 版本差异.可使用如下命令查看模块是否工作正常

```bash
 zabbix_get -s 192.168.0.120 -k docker.discovery
```

192.168.0.120 为被监控端 ip,可看到类似如下返回,表示 Docker 监控模块工作正常

```bash
{
    "data": [
        {
            "{#FCONTAINERID}": "b173b9671ea305648cb15704ce2086724578c113a65bec4724df0461e894a423",
            "{#HCONTAINERID}": "b173b9671ea3",
            "{#SCONTAINERID}": "b173b9671ea3",
            "{#SYSTEM.HOSTNAME}": "localhost.localdomain"
        },
        {
            "{#FCONTAINERID}": "fcae1c7a27f07367315de2c381bd630f1fe0e9680844074bf2c931fad5011afd",
            "{#HCONTAINERID}": "fcae1c7a27f0",
            "{#SCONTAINERID}": "fcae1c7a27f0",
            "{#SYSTEM.HOSTNAME}": "localhost.localdomain"
        }
    ]
}
```

默认模版里为使用{&#35;HCONTAINERID}为容器标签,如出现不支持或日志中出现如下

```bash
Cannot open metric file: '/sys/fs/cgroup/memory/docker/fcae1c7a27f0/memory.stat'
```

可将模版里的{&#35;HCONTAINERID}变量替换为{&#35;FCONTAINERID}即可监控
![metric](https://img.cactifans.com/wp-content/uploads/2017/12/673D309B7F27345B3542875373BF9B16.jpg)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
