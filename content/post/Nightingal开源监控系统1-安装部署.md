---
title: Nightingale开源监控系统1-安装部署
tags: [nightingale]
categories: [nightingale]
date: 2020-11-09 10:27:30
---

Nightingale 是开源的监控系统，目前 V3 版本已从监控告警系统，演化为一个运维平台，平台使用 Go 语言编写

## 系统架构

![1](https://img.cactifans.com/wp-content/uploads/2020/11/1604934002757.jpg)

## 系统组成

夜莺拆成了四个子系统，分别是：用户资源中心（RDB）、资产管理系统（AMS）、任务执行中心（JOB）、监控告警系统（MON）。下面分别介绍一下这几个子系统的设计初衷

### 用户资源中心

这是一个平台底座，所有的运维系统，都需要依赖这个，内置用户、权限、角色、组织、资源的管理。最核心的是一棵组织资源树，树节点的类别和扩展字段可以自定义，组织资源树的层级结构最简单的组织方式是：租户》项目》模块，复杂一点的组织方式：租户》组织》项目》模块》集群，组织是可以嵌套的。节点上挂两类对象，一个是人员权限，一个是资源，资源可以是各类资源，除了主机设备、网络设备，也可以是 rds 实例，redis 实例，当然，这就需要 rds、redis 的管控系统和 RDB 打通了。滴滴在做一些大的中后台商业化解决方案的时候，RDB 就是扮演了这么一个底座的角色。

### 资产管理系统

这里的资产管理系统，是偏硬件资产的管理，这个系统的使用者一般是系统部的人，资产管理类人员，应用运维相对不太关注这个系统。开源版本开放了一个主机设备的管理，大家可以二开，增加一些网络设备管理、机柜机架位的管理、配件耗材的管理等等，有了底座，上面再长出一些其他系统都相对容易。agent 安装之后，会自动注册到资产管理系统，自动采集到机器的 sn、ip、cpu、mem、disk 等信息，这些信息为了灵活性考虑，都是用 shell 采集的，上文“安装步骤”一章有提到，其中最重要的是 ip，系统中有很多设备，ip 是需要全局唯一，其他的 sn、cpu、mem、disk 等，如果无法采集成功，可以写死，shell 里直接写 echo 一个假数据即可。
每一条资产，都有一个租户的字段，代表资产归属，需要管理员去分配资产归属（修改资产的所属租户），各个租户才能使用对应的资产，分配完了之后，会出现在用户资源中心的“游离资源”菜单中，各个租户就可以把游离资源挂到资产树上去分门别类的管理使用。树节点的创建是在树上右键哈。

### 任务执行中心

用于批量跑脚本，类似 pssh、ansible、saltstack，不过不支持 playbook，大道至简，就用脚本撸吧，shell、python、perl、ruby，都行，只要机器上有解析器。因为是内置到夜莺里的，所以体系化会更好一些，和组织资源树的权限是打通的，可以控制不同的人对不同的机器有不同的权限，有些人可以用 root 账号执行，有些人只能用普通账号执行，历史执行记录都可以通过 web 页面查看审计。任务本身支持一些控制：暂停点、容忍度、单机超时时间、中途暂停、中途取消、中途 Kill 等。
一些经常要跑的脚本，可以做成模板，模板是对脚本的一种管理方式，后续就可以基于模板创建任务，填个机器列表就可以执行。比如安装 JDK，调整 TCP 内核参数，调整 ulimit 等机器初始化脚本，都可以做成模板。
开源版本的任务执行中心，可以看做是一个命令通道，后续可以基于这个命令通道构建一些场景化应用，比如机器初始化平台、服务变更发布平台、配置分发系统等。任务执行中心各类操作都有 API 对外暴露，具体可参看：router.go 我司的命令通道每周执行任务量超过 60 万，就是因为各类上层业务都在依赖这个命令通道的能力。

### 监控告警系统

这块核心逻辑和 v2 版本差别不大，监控指标分成了设备相关指标和设备无关指标，因为有些自定义监控数据的场景，endpoint 不好定义，或者 endpoint 经常变化，这种就可以使用设备无关指标的方式来处理。监控大盘做了优化，引入了更多类型的图表，但夜莺毕竟是个 metrics 监控系统，处理的是数值型时序数据，所以，最有用的图表其实就是折线图，其他类型图表，看看就好，场景较少。夜莺也可以对接 Grafana，有个专门的 DataSource 插件，Grafana 会更炫酷一些，只是，在数据量大的时候性能较差。

## 文档手册

文档：https://github.com/didi/nightingale/wiki  
代码：https://github.com/didi/nightingale

## 安装部署

### 基础环境配置

准备一台干净的 centos7 操作系统，关闭 selinux，关闭防火墙，更新系统并安装 epel 源

```
 yum update -y
 yum install epel-release -y
```

安装 mysql，redis，ngixn 等组件

```
yum install -y mariadb* redis nginx wget net-tools
```

配置组件启动并配置开机自启动

```
systemctl enable --now mariadb redis nginx
```

使用 mysql_secure_installation 命令初始化 mysql 数据库，并配置 root 密码，方便期间配置为 1234

### 系统部署

新手建议直接使用编译好的二进制程序即可,本次以安装 v3.1.6 版本

```
mkdir -p /usr/local/n9e
cd /usr/local/n9e
wget http://116.85.64.82/n9e-3.1.6.tar.gz
tar zxvf n9e-3.1.6.tar.gz
```

初始化数据库，如果你的 root 密码不为 1234，可在/usr/local/n9e/etc/mysql.yml 里修改

```
cd /usr/local/n9e/sql
mysql -uroot -p1234 < n9e_ams.sql
mysql -uroot -p1234 < n9e_hbs.sql
mysql -uroot -p1234 < n9e_job.sql
mysql -uroot -p1234 < n9e_mon.sql
mysql -uroot -p1234 < n9e_rdb.sql
```

默认配置 redis 端口为 6379，密码为空，如果默认配置不对，可以执行如下命令，看到多个配置文件里有 redis 相关配置，挨个检查修改下

```
cd /usr/local/n9e/etc
grep redis -r .
```

下载打包好的前端文件

```
cd /usr/local/n9e/
wget http://116.85.64.82/pub.tar.gz
tar zxvf pub.tar.gz
```

修改自带 nginx 配置文件/usr/local/n9e/etc/nginx.conf 里的默认前端路径

```
root         /usr/local/n9e/pub;
```

并覆盖系统默认的 nginx 配置文件

```
cp /usr/local/n9e/etc/nginx.conf /etc/nginx/nginx.conf
```

重启 nginx

```
systemctl restart nginx
```

修改/usr/local/n9e/etc/identity.yml 文件，建议 specify 修改为机器 IP

```
# 用来做心跳，给服务端上报本机ip
ip:
  specify: "172.16.66.16"
  shell: ifconfig `route|grep '^default'|awk '{print $NF}'`|grep inet|awk '{print $2}'|head -n 1

# MON、JOB的客户端拿来做本机标识
ident:
  specify: "172.16.66.16"
  shell: ifconfig `route|grep '^default'|awk '{print $NF}'`|grep inet|awk '{print $2}'|head -n 1
```

检查无误后启动 n9e 后台服务

```
cd /usr/local/n9e/
./control start all
```

如配置无误，所有服务都会正常启动。

### Web

使用浏览器访问 http://ip 即可看到 n9e 的 web 页面，默认账号：root 密码：root.2020 即可登录系统，至此安装完成。
![2](https://img.cactifans.com/wp-content/uploads/2020/11/1604933934065.jpg)
![3](https://img.cactifans.com/wp-content/uploads/2020/11/1604934600339.jpg)
![4](https://img.cactifans.com/wp-content/uploads/2020/11/1604934616368.jpg)

## Debug

如组件启动失败，建议检查 etc 下配置文件，并查看 logs 文件夹下的日志排错。

下一篇主要介绍 Nightingale 的监控告警配置。
