---
title: Zabbix 国内源使用
date: 2019-01-21 13:10:55
tags: [zabbix, zabbix4.0, 国内源]
categories: [zabbix]
---

Zabbix 官网提供了通过 yum 或 apt 方式安装 zabbix，由于国内网路原因，导致失败。其实在国内已经有很多 zabbix 的 repo 镜像站点，添加之后即可顺畅 yum 或 apt。

### 国内 zabbix 源总结

目前发现的有以下几个站点: 1.阿里巴巴开源镜像站(**推荐使用**)
地址：https://mirrors.aliyun.com/zabbix/ 2.华为开源镜像站(推荐使用)
地址：https://mirrors.huaweicloud.com/zabbix/ 3.清华大学开源软件镜像站
地址：https://mirror.tuna.tsinghua.edu.cn/zabbix/ 4.上海大学开源镜像站
地址：https://mirrors.shu.edu.cn/zabbix/

以上几个可根据自身网络情况选择使用

### 使用方法

以使用阿里巴巴开源镜像站为例子，介绍如何使用 zabbix 4.0 源

### RHEL7/CentOS7

添加源

```bash
cat <<EOF > /etc/yum.repos.d/zabbix.repo
[zabbix]
name=Zabbix Official Repository - \$basearch
baseurl=https://mirrors.aliyun.com/zabbix/zabbix/4.0/rhel/7/\$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

[zabbix-non-supported]
name=Zabbix Official Repository non-supported - \$basearch
baseurl=https://mirrors.aliyun.com/zabbix/non-supported/rhel/7/\$basearch/
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
gpgcheck=1
EOF
```

添加 gpgkey

```bash
curl https://mirrors.aliyun.com/zabbix/RPM-GPG-KEY-ZABBIX-A14FE591 \
-o /etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

curl https://mirrors.aliyun.com/zabbix/RPM-GPG-KEY-ZABBIX \
-o /etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
```

添加之后即可使用

```bash
yum makecache -y
yum install zabbix-<...>
```

进行流畅的安装 zabbix 各个组件

### Ubuntu 16.04

```bash
cat > /etc/apt/sources.list.d/zabbix.list << EOF
deb https://mirrors.aliyun.com/zabbix/zabbix/4.0/ubuntu xenial main
deb-src https://mirrors.aliyun.com/zabbix/zabbix/4.0/ubuntu xenial main
EOF
```

添加 key

```bash
curl -o - "https://mirrors.aliyun.com/zabbix/zabbix-official-repo.key" | apt-key add -
```

更新源，安装相关包

```bash
apt-get update
apt-get install -y zabbix-<...>
```

### Debian 9

添加源

```bash
cat > /etc/apt/sources.list.d/zabbix.list << EOF
deb https://mirrors.aliyun.com/zabbix/zabbix/4.0/debian stretch main
deb-src https://mirrors.aliyun.com/zabbix/zabbix/4.0/debian stretch main
EOF
```

添加 key

```bash
curl -o - "https://mirrors.aliyun.com/zabbix/zabbix-official-repo.key" | apt-key add -
```

更新源，安装相关包

```bash
apt-get update
apt-get install -y zabbix-<...>
```

不同源修改相应地址即可。
