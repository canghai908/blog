---
title: Zabbix5.2版本新特性-机密信息外部存储
tags: [zabbix, valut]
categories: [zabbix]
date: 2020-11-23 0:27:30
---

zabbix 5.2 版本于 2020 年 10 月 26 日正式发布！早在 9 月 26 的 zabbix 中国大会上 Alexei Vladishev 就介绍了 5.2 版本的很多特性，很值得期待，如今 5.2 版本正式发布了，本次主要介绍 5.2 的安全特性。

## 特性介绍

Alexei Vladishev 在 zabbix 的安全上做了很大的改进，如支持连接加密，保密宏等。现在 zabbix 的所有连接都可以配置为加密模式。5.2 版本引入了 HashCorp Valut 来保存一些机密信息到外部存储。HashiCorp Vault 的 口号 是 A Tool for Managing Secrets，这个口号很好的描述了该产品的定位。 HashiCorp 是一家专注于基础设施解决方案的公司，业务范围涵盖软件开发中的部署、运维、安全等方面。新版本中很多敏感信息可保存在 HashCorp Valut，而不保存在 zabbix 数据库里。如宏信息，数据库连接信息，密码，加密的 key 等。这进一步加强了 zabbix 的安全性。对于一些场景特别适用。

## 基础环境

以下为基础环境介绍

| 环境           | 版本                   |
| :------------- | :--------------------- |
| 操作系统       | Centos 8.2.2004 x86_64 |
| 数据库         | MariaDB 10.3.17        |
| HashCorp Valut | HashCorp Valut v1.5.5  |
| Zabbix         | Zabbix 5.2.1           |

使用 yum 方案安装 zabbix，使用二进制方式安装 HashCorp Valut

## HashCorp Valut

### 安装

主要介绍 Valut 的安装及基本配置，官网下载 Valut 安装包，并解压

```
wget https://releases.hashicorp.com/vault/1.5.5/vault_1.5.5_linux_amd64.zip
unzip vault_1.5.5_linux_amd64.zip
chown root:root vault
mv vault /usr/local/bin/
```

安装命令行自动补全

```
vault -autocomplete-install
complete -C /usr/local/bin/vault vault
```

安全配置

```
setcap cap_ipc_lock=+ep /usr/local/bin/vault
useradd --system --home /etc/vault --shell /bin/false vault
```

配置 systemd 启动文件

```
touch /etc/systemd/system/vault.service
```

内容如下

```
[Unit]
Description="HashiCorp Vault - A tool for managing secrets"
Documentation=https://www.vaultproject.io/docs/
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault/vault.hcl
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/vault server -config=/etc/vault/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=60
StartLimitIntervalSec=60
StartLimitBurst=3
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

官方推荐使用 consul 来作为 vault 的后端数据存储，为配置简便，本次使用本地文件存储作为 vault 的数据存储。
创建对应目录及配置文件

```
mkdir --parents /etc/vault.d
touch /etc/vault.d/vault.hcl
chown --recursive vault:vault /etc/vault
chmod 640 /etc/vault.d/vault.hcl
mkdir /opt/vault_data
chown -R vault:vault /opt/vault_data/
```

vault 配置文件为 vault.hcl，vault 可配置 ssl 证书，可以自己生成

```
disable_cache = true
disable_mlock = true
ui = true
listener "tcp" {
  tls_disable = 0
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault/vault.cactifans.com.pem"
  tls_key_file  = "/etc/vault/vault.cactifans.com.key"
}
storage "file" {
  path = "/opt/vault_data"
}
api_addr = "https://valut.cactifans.com:8200"
```

启动 vault

```
systemctl enable vault
systemctl start vault
```

确保 vault 启动。

### 初始化

第一次启动 vault 之后需要初始化，使用以下命令进行初始化

```
export VAULT_ADDR=https://vault.cactifans.com:8200
vault operator init
```

会输出如下

```
Unseal Key 1: 5D3jfaZRj4ifk4Dz3hzSg4h2ts/yw65JDfe4mp6gkZr3
Unseal Key 2: U4cAMxRyGtCljAggzsV8zrGplRWtDUrAE4dwkrZ+PGea
Unseal Key 3: Zh+ZmvvaQ8LTwSL7q8uVTjHu26+sSfnbOBsewe71NTvI
Unseal Key 4: c/AVfHxALC9cRant4jegLIDEdqKgbRmW+2DJvoHr0pdf
Unseal Key 5: efjVPwZbk7R8V2Upsci8B4GanoYxsGnCOQwycBCv69IV

Initial Root Token: s.tYt0d3k55jB6sdTnmJvUY6MV

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

存储以上几个 key 及 token 到文本，以后不会再次显示。
每次 vault 启动之后都要进行解封，才能进行数据的读取写入。输入三次以下命令，输入后会提示要输入 key，挑选之前 5 个 key 中的三个，输入，即可解封。

```
vault operator unseal
```

查看 vault 状态

```
vault status
```

状态

```
[root@vault ~]# vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.5.5
Cluster Name    vault-cluster-fe2b21f4
Cluster ID      b04d080b-662e-9a20-0699-56f6fe53a992
HA Enabled      false
```

确保 Sealed 为 false 状态,否则无法操作数据。

## zabbix 安装

按照官网文档使用 yum 方式安装 zabbix。创建好 zabbix 数据库用户导入 sql 文件后，在 vault 中使用以下命令创建 zabbix 数据库连接信息，假如 zabbix 数据库用户为 zabbix，密码为 password

```
export VAULT_ADDR=https://vault.cactifans.com:8200
vault secrets enable -path=zabbix kv-v2
vault kv put zabbix/database username=zabbix password=password
```

查看使用以下命令查看 kv

```
vault kv get zabbix/database
```

结果

```
[root@vault ~]# vault kv get zabbix/database
====== Metadata ======
Key              Value
---              -----
created_time     2020-11-13T05:45:27.927834884Z
deletion_time    n/a
destroyed        false
version          2

====== Data ======
Key         Value
---         -----
password    password
username    zabbix
```

可以看到已经配置成功。
这里再创建一个名 macros 的 path，用于存储 zabbix 的宏。这里创建一个 username 和 password 的 2 个 key，username=monitor，password=password 后续介绍使用方法。

```
vault kv put zabbix/macros username=monitor password=zabbix@2020
```

查看 kv

```
[root@vault ~]# vault kv get zabbix/macros
====== Metadata ======
Key              Value
---              -----
created_time     2020-11-15T08:15:39.462966496Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    zabbix@2020
username    monitor
```

HashiCorp Vault 有严格的权限访问控制，这里为 zabbix 配置对应的访问权限，并生成对应的访问 Token。Vault 的策略可以先写成 hcl 文件，再导入即可。这里主要配置 2 个策略
| 策略文件 | 策略 |
| :-------- |: --------|
| zabbix-ui.hcl | 可读取 zabbix/database |  
| zabbix-server.hcl | 可读取 zabbix/database 和读取 zabbix/macros|

创建策略文件/etc/vault/zabbix-ui.hcl 内容如下

```
path "zabbix/data/database" {
  capabilities = ["list","read"]
}
```

创建策略文件/etc/vault/zabbix-server.hcl 内容如下

```
path "zabbix/data/database" {
  capabilities = ["list","read"]
}
path "zabbix/data/macros" {
  capabilities = ["list","read"]
}
```

写入策略

```
vault policy write zabbix-ui-policy zabbix-ui.hcl
vault policy write zabbix-server-policy zabbix-server.hcl
```

生成 token

```
vault token create -policy=zabbix-ui-policy
vault token create -policy=zabbix-server-policy
```

记录 token。

```
[root@vault vault]# vault token create -policy=zabbix-ui-policy
---                  -----
token                s.Op7YWWK4P0tMVKwpneQbR8wT
token_accessor       6nw5I3DtoTAcLWKPXss8PfsS
token_duration       768h
token_renewable      true
token_policies       ["default" "zabbix-ui-policy"]
identity_policies    []
policies             ["default" "zabbix-ui-policy"]
[root@vault vault]# vault token create -policy=zabbix-server-policy
Key                  Value
---                  -----
token                s.XdcVY5wf4kTLxTVrKYJ98Lwu
token_accessor       6SCyEeWmSFGea6jlQjzoRgGk
token_duration       768h
token_renewable      true
token_policies       ["default" "zabbix-server-policy"]
identity_policies    []
policies             ["default" "zabbix-server-policy"]
```

使用以下命令可验证配置是否正确

```
curl -s --header "X-Vault-Token: s.XdcVY5wf4kTLxTVrKYJ98Lwu" https://vault.cactifans.com:8200/v1/zabbix/data/database |jq
```

Token 为刚才创建,如果正确会显示如下

```
{
  "request_id": "85a25acd-a7ac-ebbf-1f78-5b59b0d3a28a",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": {
    "data": {
      "password": "password",
      "username": "zabbix"
    },
    "metadata": {
      "created_time": "2020-11-13T05:45:27.927834884Z",
      "deletion_time": "",
      "destroyed": false,
      "version": 2
    }
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```

## Zabbix server 配置 Vault

修改 zabbix server 配置文件/etc/zabbix/zabbix_server.conf ,注释

```
#DBUser=zabbix
```

文件末尾配置 Vault 信息

```
VaultToken=s.XdcVY5wf4kTLxTVrKYJ98Lwu
VaultURL=https://vault.cactifans.com:8200
VaultDBPath=zabbix/database
```

VaultToken 为 zabbix-server 策略生成的 token，VaultDBPath 为 zabbix/database
重启 zabbix server 等组件

```
systemctl restart zabbix-server zabbix-agent httpd php-fpm
systemctl enable zabbix-server zabbix-agent httpd php-fpm
```

检查日志，如果没有报错表示与 Vault 通信正常。

## Zabbix Web 配置 Vault

打开 zabbix web 安装页面，到此页面输入 vault 信息
![1](https://img.cactifans.com/wp-content/uploads/2020/11/1605250239409.jpg)
path 为 secret/zabbix/database，token 为 zabbix-ui 策略生成的 token，直接点击下一步，如提示错误可能可能是地址或者策略配置文件，如连接 ok 会到下一步
![2](https://img.cactifans.com/wp-content/uploads/2020/11/1605250279808.jpg)

## Vault 存储宏

新版本可将 zabbix 宏存储在 Vault 中，之前已在 Vault 创建一个名为 macros 的 path，后期可使用以下命令创建需要的 macros,直接写在后面即可，如添加一个 key 为 token，value 为 123456

```
vault kv put zabbix/macros username=monitor password=zabbix@2020 token=123456
```

查看

```
vault kv get zabbix/macros
```

结果

```
[root@vault ~]# vault kv get zabbix/macros
====== Metadata ======
Key              Value
---              -----
created_time     2020-11-15T08:22:36.667284971Z
deletion_time    n/a
destroyed        false
version          2

====== Data ======
Key         Value
---         -----
password    zabbix@2020
token       123456
username    monitor
```

说明已添加完成。

## Vault 宏使用

下面介绍如何在 zabbix 中如何使用 vault 保存的宏。例如使用 ssh agent 采集时需要输入机器的账号和密码，这里可使用 vault 存储账号和密码信息。下面主要介绍此场景。在主机上执行以下命令添加系统账号，用于 ssh 监控

```
useradd monitor
passwd monitor
```

系统账号为 monitor，并配置密码为 zabbix@2020
在主机上配置 2 个宏,分别为 {$USERNAME}  和  {$PASSWORD}。注意宏的类型为 Vault secret 并配置 path 为 zabbix/macros:username 和 zabbix/macros:password
![3](https://img.cactifans.com/wp-content/uploads/2020/11/1605429755033.jpg)
并创建如下 ssh agent 类型的 item，
![4](https://img.cactifans.com/wp-content/uploads/2020/11/1605429838589.jpg)
username 及 password 字段配置为刚才添加宏名称，执行命令为 free -g。配置之后点击保存。过一会可在最数据里查看，获取正常说明宏调用 ok
![5](https://img.cactifans.com/wp-content/uploads/2020/11/1605429861233.jpg)
历史数据
![6](https://img.cactifans.com/wp-content/uploads/2020/11/1605429893358.jpg)

## Tips

HashiCorp Vault 有 Web 页面，可使用浏览器访问，默认端口为 8200，可使用 Token 登录进行操作。
![7](https://img.cactifans.com/wp-content/uploads/2020/11/1605430950561.jpg)

## 结语

以上为新版本配合 HashCorp Valut 的使用。HashCorp Valut 的使用大大加强了 zabbix 的安全性，同时也方便了各种敏感信息的统一管理和使用。
