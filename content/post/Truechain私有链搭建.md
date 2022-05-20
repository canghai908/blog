---
title: Truechain私有链搭建
date: 2018-09-21 16:35:35
tags: [truechain]
categories: [truechain]
---

### 环境介绍

| 名称         | 主机名 | 操作系统      | IP           |
| ------------ | ------ | ------------- | ------------ |
| 委员会节点 1 | node01 | Centos7.5 x64 | 172.16.66.10 |
| 委员会节点 2 | node02 | Centos7.5 x64 | 172.16.66.11 |
| 委员会节点 3 | node03 | Centos7.5 x64 | 172.16.66.16 |
| 委员会节点 4 | node04 | Centos7.5 x64 | 172.16.66.17 |

### 初始化

以下命令在各个节点执行
安装 docker 环境并启动 docker

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
systemctl start docker
systemctl enable docker
```

下载镜像

```
docker pull registry.cn-hangzhou.aliyuncs.com/truechain_space/truechain_image:latest
```

更改镜像 tag 为 etrue

```
docker tag registry.cn-hangzhou.aliyuncs.com/truechain_space/truechain_image getrue
```

### 配置文件

在 node01 上建立目录，并建立几个初始化脚本

```bash
mdiir -p /opt/truechain
```

并建立一个传世快初始化配置文件

```
cd /opt/truechain
touch gensis.json
```

文件内容如下

```json
{
  "config": {
    "chainId": 10,
    "homesteadBlock": 0,
    "eip155Block": 0,
    "eip158Block": 0
  },
  "alloc": {
    "0x7c357530174275dd30e46319b89f71186256e4f7": {
      "balance": 90000000000000000000000
    },
    "0x4cf807958b9f6d9fd9331397d7a89a079ef43288": {
      "balance": 90000000000000000000000
    },
    "0x04d2252a3e0ca7c2aa81247ca33060855a34a808": {
      "balance": 90000000000000000000000
    },
    "0x05712ff78d08eaf3e0f1797aaf4421d9b24f8679": {
      "balance": 90000000000000000000000
    },
    "0x764727f61dd0717a48236842435e9aefab6723c3": {
      "balance": 90000000000000000000000
    },
    "0x764986534dba541d5061e04b9c561abe3f671178": {
      "balance": 90000000000000000000000
    },
    "0x0fd0bbff2e5b3ddb4f030ff35eb0fe06658646cf": {
      "balance": 90000000000000000000000
    },
    "0x40b3a743ba285a20eaeee770d37c093276166568": {
      "balance": 90000000000000000000000
    },
    "0x9d3c4a33d3bcbd2245a1bebd8e989b696e561eae": {
      "balance": 90000000000000000000000
    },
    "0x35c9d83c3de709bbd2cb4a8a42b89e0317abe6d4": {
      "balance": 90000000000000000000000
    }
  },

  "committee": [
    {
      "address": "0x76ea2f3a002431fede1141b660dbb75c26ba6d97",
      "publickey": "0x04044308742b61976de7344edb8662d6d10be1c477dd46e8e4c433c1288442a79183480894107299ff7b0706490f1fb9c9b7c9e62ae62d57bd84a1e469460d8ac1"
    }
  ],
  "coinbase": "0x0000000000000000000000000000000000000000",
  "difficulty": "0x100",
  "extraData": "",
  "gasLimit": "0x5400000",
  "nonce": "0x0000000000000042",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp": "0x00"
}
```

为了方便操作，我们把初始化做成一个脚本名为 init_gensis.sh 内容如下

```bash
docker run -v $PWD:/truechain-engineering-code -v $PWD/data:/truechain-engineering-code/data -it etrue --datadir /truechain-engineering-code/data init /truechain-engineering-code/genesis.json
```

赋予可执行权限

```bash
chmod a+x /opt/truechain/init_gensis.sh
```

在/opt/truechain 执行初始化

```
./init_gensis.sh
```

执行之后会生成出现如下日志

```
[root@www truechain]# ./init_gensis.sh
INFO [09-21|07:15:04.974] Maximum peer count                       ETRUE=25 LES=0 total=25
INFO [09-21|07:15:04.975] Committee Node info:                     publickey=0473f661ec8e634594216daf828d9fe728ea1660db13916a656aa348974518772e2c21b25ee9611156fcccaf64db79cfab32eaf2b326049741b91c8144290b92c3 ip= port=10080 election=false singlenode=false
INFO [09-21|07:15:04.975] Allocated cache and file handles         database=/truechain-engineering-code/data/getrue/chaindata cache=16 handles=16
INFO [09-21|07:15:04.987] Persisted trie from memory database      nodes=15 size=2.17kB time=94.532µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [09-21|07:15:04.992] Successfully wrote genesis state         database=chaindata                                         hash=285dec…336666 snail=84ecbf…86347f
INFO [09-21|07:15:04.992] Allocated cache and file handles         database=/truechain-engineering-code/data/getrue/lightchaindata cache=16 handles=16
INFO [09-21|07:15:05.007] Persisted trie from memory database      nodes=15 size=2.17kB time=67.855µs gcnodes=0 gcsize=0.00B gctime=0s livenodes=1 livesize=0.00B
INFO [09-21|07:15:05.010] Successfully wrote genesis state         database=lightchaindata                                         hash=285dec…336666 snail=84ecbf…86347f
```

记录下 publickey，并用以下命令查看 privatekey

```bash
cat data/getrue/bftkey
```

并记录下 privatekey，删除目录下的 data 文件，再次执行初始化 init_gensis.sh 脚本，生成一对 publickey 和 privatekey ，按照方法生成 4 对 key，并填入 gensis.json 文件，最终的 gensis.json 文件格式如下

```bash
{
"config": {
   "chainId": 10,
   "homesteadBlock": 0,
   "eip155Block": 0,
   "eip158Block": 0
 },
 "alloc":{
   "0x7c357530174275dd30e46319b89f71186256e4f7" : { "balance" : 90000000000000000000000},
   "0x4cf807958b9f6d9fd9331397d7a89a079ef43288" : { "balance" : 90000000000000000000000},
   "0x04d2252a3e0ca7c2aa81247ca33060855a34a808" : { "balance" : 90000000000000000000000},
   "0x05712ff78d08eaf3e0f1797aaf4421d9b24f8679" : { "balance" : 90000000000000000000000},
   "0x764727f61dd0717a48236842435e9aefab6723c3" : { "balance" : 90000000000000000000000},
   "0x764986534dba541d5061e04b9c561abe3f671178" : { "balance" : 90000000000000000000000},
   "0x0fd0bbff2e5b3ddb4f030ff35eb0fe06658646cf" : { "balance" : 90000000000000000000000},
   "0x40b3a743ba285a20eaeee770d37c093276166568" : { "balance" : 90000000000000000000000},
   "0x9d3c4a33d3bcbd2245a1bebd8e989b696e561eae" : { "balance" : 90000000000000000000000},
   "0x35c9d83c3de709bbd2cb4a8a42b89e0317abe6d4" : { "balance" : 90000000000000000000000}
 },

 "committee":[
   {
     "address": "0xd565db0d65894038ecfbe587e9bc836a1da7a97d",
     "publickey": "0x04d16938b4333da470f7c47674a3b2e728e49f97cc76bd7cc51f6ae29bafaf0fb4167222aea6dab04fab5828a127ec83700287856e305f01ec5f02918e2f5e622b",
     "privatekey": "4eed8ce61acc1c55e7f8fb608bc2bfef7f6cdbd40313c6ccd16de04362195e43"
   },
   {
     "address": "0x3f5b510beaa9b25b79ec9bfe08bd2b5e9fed033d",
     "publickey": "0x04d31f1a2bf1db8139f0bf4045a7be96872bb9ed2d187094d04473feda50f4f74a85a227fd943fe31986de04f44420f19c0c3d7197d7d196e9720eafa223597840",
     "privatekey": "57078f59c8c86e80a1cce28067d2dde54b8d8eb3352d29d50bb6364ad0ed192f"
   },
   {
     "address": "0x84e6f94d78759295dac2bf3d7f9d809f60992a62",
     "publickey": "0x04fc474235c9554614d308cf0ad93104710953901057d9d728879865e3ba8b49b88d1977294f7878596bf9f03af7c841100239a4f57c1bc45a177a35d00de5faac",
     "privatekey": "3df7b465478b5154ee986957198fddeee02206de3c78763875fc55bfcf11b389"
   },
   {
     "address": "0xf88fb95e9debdfcd2180b552e88fd0cde5c0c322",
     "publickey": "0x0405e965348dfe9a7b29d89d0b15b75f5b1caf7129bef5159a79fd2c62701553387eb130e377bd57ee3a2b25a4a7ae0768a9d2c77f197bda4badc5cb70a1ae943a",
     "privatekey": "cb4b9f582f98b1ab00ceacfc85d0d3a66127b9918ff84baff9e4a210643378e1"
   }
 ]
,
 "coinbase"   : "0x0000000000000000000000000000000000000000",
 "difficulty" : "0x100",
"extraData"  : "",
 "gasLimit"   : "0x5400000",
 "nonce"      : "0x0000000000000042",
 "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
 "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
 "timestamp"  : "0x00"
}
```

之后再创建启动委员会的脚本 start_work.sh，内容如下

```bash
docker run -v $PWD:/truechain-engineering-code -p 30303:30303 -p 10080:10080 -p 8545:8545 -it  etrue  --datadir /truechain-engineering-code/data  --bftkeyhex "4eed8ce61acc1c55e7f8fb608bc2bfef7f6cdbd40313c6ccd16de04362195e43" --election --bftip "0.0.0.0" --bftport 10080 --port 30303 --networkid 10 --rpc --rpcaddr 127.0.0.1 --rpcport 8545 --rpcapi "db,etrue,net,web3,personal,admin,miner"  --verbosity 1 console
```

清理掉 data 目录

```bash
rm -rf /opt/truechain/data
```

至此，配置文件及脚本已修改完毕，把整个 truechain 文件夹传到其他 3 个机器 ###配置节点配置

```
scp -r /opt/truechain/ root@172.16.66.11:/opt
scp -r /opt/truechain/ root@172.16.66.16:/opt
scp -r /opt/truechain/ root@172.16.66.17:/opt
```

复制之后，修改 node02.node03,node04 节点的 start_work.sh 文件，修改 bftkeyhex 为对应的 privatekey.。
node02 为第 2 对 key 的 privatekey
node03 为第 3 对 key 的 privatekey
node04 为第 4 对 key 的 privatekey
再各个节点先执行初始化脚，再执行委员会脚本

```
cd  /opt/truechain
./init_gensis.sh
./start_work.sh
```

启动之后，可以看到如下提示，表示启动成功

```
[root@monitor truechain]# ./start_work.sh
fastQueue>>>>>>>>>>>>> 0xc4202408e8
Welcome to the Getrue JavaScript console!

instance: Getrue/v0.7.2-unstable/linux-amd64/go1.10.4
 modules: admin:1.0 debug:1.0 etrue:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

>
```

在各个节点使用 admin.nodeInfo.enode 查看节点信息，如下

```
> admin.nodeInfo.enode
"enode://6ff04de8fbb85dc80f1604062057da6e90390e4f256691e79541eee57a8e3d004917c45bb3b1476258a28c9215c5c43feff8dc67ee3f5f91cbc2b3ec738715ac@[::]:30303"
>
```

记录下节点信息，修改后面的[::]为节点的实际 ip,并编辑成以下形式的命令

```
admin.addPeer("enode://9eec6aa38162cb94c87f62ebc5492c43ce6e8dd039e6dfc55be7521c9033c7bfb14be7889efaa22697ae2917ebb4ad0f65dc9033956606d23959e742c5c39295@172.16.66.10:30303")

admin.addPeer("enode://1c14e786fd1876f112464cb9ba5a30256cce039e79f802a4c24a649b9cfac54f6f4d0889c36804a1efba01ac5e9ef062511ed4c22580a1fc15fbbabc98f59082@172.16.66.11:30303")

admin.addPeer("enode://8f06c0080e8d3944d1f356ddc5d401d14afe159ec16f251d1ec39236703638b23c40e9a1397e2e5a95cf8f2bf62fbeef514f43664f4c33f45d2552b620780471@172.16.66.16:30303")

admin.addPeer("enode://560831be3d76f2526e56680e4e573d4e1c4e02a2c6edf866e71857a67e03131b59dee8e724978162982b522468609b9dd6ddffa1b7693c459148f3a3265100c3@172.16.66.17:30303")
```

以上节点信息根据自己的实际情况修改，后面为节点的实际 ip，端口为 30303,复制修改好的信息，张贴到各个节点的终端，即可完成添加节点工作。效果如下

```
> admin.addPeer("enode://6ff04de8fbb85dc80f1604062057da6e90390e4f256691e79541eee57a8e3d004917c45bb3b1476258a28c9215c5c43feff8dc67ee3f5f91cbc2b3ec738715ac@172.16.66.10:30303")
true
>
> admin.addPeer("enode://c1fcfe59f9224171b66d944687a69f2b0ac69386b5d131c03fddf743727976c3a43bea9dcd04d443c25f2043c8470198bac80357789641c215da0b5219e4d3f3@172.16.66.11:30303")

true
>
> admin.addPeer("enode://d4e47c320694aaf14f58889752db148447a15511cf8d85514415ea84c7e07b13f1cf0ac81b17ca322fef676777dcb1bd523781ab1811e81f7ed8c23f786b5998@172.16.66.16:30303")
admin.addPeer("enode://aaa33e05c3d100ad479f1b412341f7c66677673424774ff47c6b9e24e2ef607a6817c4ff186e901b6a008aftrue712b5a7477
cc69dd72342900b91c>
> admin.addPeer("enode://aaa33e05c3d100ad479f1b412341f7c66677673424774ff47c6b9e24e2ef607a6817c4ff186e901b6a008af712b5a7477cc69dd72342900b91c26d2e4ce72932@172.16.66.17:30303")
true
>
```

完成之后，在各个节点执行 admin.peers 即可看到都有 3 个节点

```bash
> admin.peers
[{
    caps: ["etrue/63"],
    id: "6ff04de8fbb85dc80f1604062057da6e90390e4f256691e79541eee57a8e3d004917c45bb3b1476258a28c9215c5c43feff8dc67ee3f5f91cbc2b3ec738715ac",
    name: "Getrue/v0.7.2-unstable/linux-amd64/go1.10.4",
    network: {
      inbound: true,
      localAddress: "172.17.0.2:30303",
      remoteAddress: "172.16.66.10:59248",
      static: false,
      trusted: false
    },
    protocols: {
      etrue: {
        difficulty: 256,
        head: "0x84ecbf38ddba4f544bd9f047db056bdae871dd16dce1f6ae9295c1c9ef86347f",
        version: 63
      }
    }
}, {
    caps: ["etrue/63"],
    id: "aaa33e05c3d100ad479f1b412341f7c66677673424774ff47c6b9e24e2ef607a6817c4ff186e901b6a008af712b5a7477cc69dd72342900b91c26d2e4ce72932",
    name: "Getrue/v0.7.2-unstable/linux-amd64/go1.10.4",
    network: {
      inbound: true,
      localAddress: "172.17.0.2:30303",
      remoteAddress: "172.16.66.16:39450",
      static: false,
      trusted: false
    },
    protocols: {
      etrue: {
        difficulty: 256,
        head: "0x84ecbf38ddba4f544bd9f047db056bdae871dd16dce1f6ae9295c1c9ef86347f",
        version: 63
      }
    }
}, {
    caps: ["etrue/63"],
    id: "d4e47c320694aaf14f58889752db148447a15511cf8d85514415ea84c7e07b13f1cf0ac81b17ca322fef676777dcb1bd523781ab1811e81f7ed8c23f786b5998",
    name: "Getrue/v0.7.2-unstable/linux-amd64/go1.10.4",
    network: {
      inbound: true,
      localAddress: "172.17.0.2:30303",
      remoteAddress: "172.16.66.17:49654",
      static: false,
      trusted: false
    },
    protocols: {
      etrue: {
        difficulty: 256,
        head: "0x84ecbf38ddba4f544bd9f047db056bdae871dd16dce1f6ae9295c1c9ef86347f",
        version: 63
      }
    }
}]
```

### Tips

1.退出 Docker 进程
使用 Ctrl+q 或 Ctrl+p 2.退出 docker 如何进入
使用 docker attach c37e91c50935 进入 docker，c37e91c50935 为容器 id，可使用 docker ps -a 命令查看

```bash
[root@localhost truechain]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                                   NAMES
c37e91c50935        etrue               "getrue --datadir /t…"   8 minutes ago       Up 2 seconds        0.0.0.0:8545->8545/tcp, 0.0.0.0:10080->10080/tcp, 0.0.0.0:30303->30303/tcp, 30303/udp   elegant_franklin
```
