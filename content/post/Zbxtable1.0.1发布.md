---
title: Zbxtable 1.0.1发布
tags: [zbxtalbe]
categories: [zbxtalbe]
date: 2020-07-27T01:43:07+08:00
---

zbxtable 发布之后收到很多热心朋友使用的反馈和建议，经过整理和调整 ZbxTable 1.0.1 版本发布了。

## 更新内容

**Bug 修复**

- 修复 zbxtable 组件与 ms-agent 组件日志权限问题
- 修复 zbxtable 调用 zabbix web，如果 web 服务器为 http 请求 400，图形显示异常问题。
- 修复 zbxtable 打包脚本 control 问题。
- 更新配置文件 zabbix server 提示为 zabbix web，更便于理解。
- 移除 zbxtable 配置文件 appurl 配置，便于图形调用和显示。

**特性**

- 添加 yum 安装源，可方便安装和更新组件
- 添加 centos6 和 8 支持

## 安装方式

为方便后续组件的安装和更新，zbxtable 建立了 yum 源，目前支持 centos6/7/8 系统，rpm 包采用 GPG 签名。添加 yum 源之后，可使用 yum 进行组件的安装与升级。

## 1 添加 yum 源

CentOS 6.x x86_64

```
rpm -Uvh https://repo.cactifans.com/zbxtable/1.0/rhel/6/x86_64/zbxtable-release-1.0-1.el6.noarch.rpm
yum clean all
```

CentOS 7.x x86_64

```
rpm -Uvh https://repo.cactifans.com/zbxtable/1.0/rhel/7/x86_64/zbxtable-release-1.0-1.el7.noarch.rpm
yum clean all
```

CentOS 8.x x86_64

```
rpm -Uvh https://repo.cactifans.com/zbxtable/1.0/rhel/8/x86_64/zbxtable-release-1.0-1.el8.noarch.rpm
dnf clean all
```

## 2 安装组件

如果已安装旧的组件则会进行更新。**更新前务必备份配置文件**  
安装 Zbxtable

```
yum install zbxtable -y
```

安装 Zbxtable-Web

```
yum install zbxtable-web -y
```

安装 ms-agent

```
yum install ms-agent -y
```

## 使用文档

考虑后续文档及问题汇总，后续使用及配置可查看  
Zbxtable 网站: [https://zbxtable.cactifans.com/](https://zbxtable.cactifans.com/)  
帮助文档：[https://zbxtable.cactifans.com/docs/](https://zbxtable.cactifans.com/docs/)  
发布公告：[https://zbxtable.cactifans.com/blog/releases/](https://zbxtable.cactifans.com/blog/releases/)

## 公众号

![微信](https://img.cactifans.com/wp-content/uploads/2020/07/257B5CA9-DCBD-43FC-A2DC-57EBC50390AE.png)

## QQ 群

![1](https://img.cactifans.com/wp-content/uploads/2020/07/9E2AB577-95F9-415D-9EFB-8104112F9EE6.png)
