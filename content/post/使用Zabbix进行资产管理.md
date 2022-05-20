---
title: "使用Zabbix进行资产管理"
date: 2022-04-20 12:22:00
tags: [资产管理]
categories: [zabbix]
---

zabbix作为企业级开源监控平台资产管理也是其内置的功能之一，使用自带的资产管理可自动采集信息，在中小企业中可完全代替人肉Excel实现简单的资产管理
## 1.初试资产
zabbix的资产功能在很早之前就已经存在，可进行简单的资产管理。本次以zabbix6.0版本为例子；
在主界面上点击Inventory-->overview可根据资产类型搜索对应设备，
![1](https://img.cactifans.com/wp-content/uploads/2022/04/20220420145349.png)
点击Host可查看已绑定资产的设备
![2](https://img.cactifans.com/wp-content/uploads/2022/04/20220420162718.png)
那这里的资产是如何绑定到主机的呢？对于此问题是很多人的疑惑，此配置可通过模板批量配置也可手动录入。
## 2.资产模式
zabbix的资产配置有三种模式，分别为：Disabled，Manual，Automatic
- Disabled：禁用主机资产管理功能
- Manual：通过手动添加相关资产信息
- Automatic：通过关联相关的Item指标，自动填充资产信息

点击Configuration-->Hosts，任意选择一个Hosts，点击Inventory标签，即可看到当前主机的资产配置模式，默认为禁用。
![3](https://img.cactifans.com/wp-content/uploads/2022/04/20220420163302.png)
zabbix 提供70个资产字段，可完全满足对主机资产的管理。
### 3.映射指标
通常情况下建议使用自动模式，主机Invertory模式可批量开启配置，点击Configuration-->Hosts，选中多个主机点击Mass update按钮，Inventory mode选择Automatic即可，此页面还可对主机的Inventory 指标进行批量配置。
![4](https://img.cactifans.com/wp-content/uploads/2022/04/20220420184634.png)
开启Automatic模式后，可绑定指定的Item到对应的Inventory字段。一般建议按照模板来绑定，做好指标的对应关系。
## 4.典型应用
在实际应用中，往往需要对交换机、Linux操作系统、Windows操作系统等不同类型的设备进行采集固定指标，比如设备CPU使用率、内存使用率、序列号等，由于不同类型的设备可能绑定不同类型的模板，而对应的指标又是不同的Item或者Key，因此无法实现统一的方法获取。此场景下可通过绑定到指定的Inventory字段，通过提取主机对应的Inventory字段即可获取。在配置Inventory字段映射之前，建议做好配置对应表。例如：

| Inventory字段|    Item字段 | 指标含义|
| :-------- | :--------| :-- |
| software_app_a  | CPU utilization 	 |  CPU使用率  |
| software_app_b	 | 	Memory utilization  	 |  内存使用率 |
| software_app_c  | Total memory in Bytes	 | 总内存  |
|...... | ......	 | ......  |

可将不同模板的指标绑定到同一个Inventory字段。以绑定CPU utilization为例子，点击Configuration-->Templates选择Linux by Zabbix agent模板，点击CPU utilization指标，在Populates host inventory field字段下拉选择对应的Inventory字段，点击Update即可。
![5](https://img.cactifans.com/wp-content/uploads/2022/04/20220420193827.png)
绑定之后，如果主机绑定了这个模板，并开启Inventory模式为Automatic，即可填充对应主机的CPU使用率指标到主机的Inventory字段，并且此数值会根据采集指标的变化而变化。此方法可大大简化指标的统一，如做CPU使用率Top指标时可直接对比即可，不用从具体的Item指标获取，也不用关心具体的Item及Key。
### 5.原生改造
zabbix自带的Inventory字段名称可能不适用于你的环境，可通过简单的修改达到显示的自定义。如需要将Inventory的Type字段修改为HostType，可编辑zabbix前端的include/hosts.inc.php文件
```
vi include/hosts.inc.php
```
搜索getHostInventories字段
```
function getHostInventories($orderedByTitle = false) {
        /*
         * WARNING! Before modifying this array, make sure changes are synced with C
         * C analog is located in function DBget_inventory_field() in src/libs/zbxdbhigh/db.c
         */
        $inventoryFields = [
                1 => [
                        'nr' => 1,
                        'db_field' => 'type',
                        'title' => _('Type')
                ],
                2 => [
                        'nr' => 2,
                        'db_field' => 'type_full',
                        'title' => _('Type (Full details)')
                ],
                3 => [
                        'nr' => 3,
                        'db_field' => 'name',
                        'title' => _('Name')
                ],
                4 => [
                        'nr' => 4,
                        'db_field' => 'alias',
                        'title' => _('Alias')
                ],
```
将
```
'title' => _('Type')
```
修改为
```
'title' => _('HostType')
```
保存文件页面发现已经修改成功。
![5](https://img.cactifans.com/wp-content/uploads/2022/04/20220420195610.png)
这里只是修改页面显示的标题，并不修改数据库字段，通过此方法修改后，如后期对zabbix进行升级后要重新修改。
### 6.API应用
在zabbix API中Inventory对应的操作并没有提供独立的API，而是通过zabibx的Host api提供，字段介绍[https://www.zabbix.com/documentation/current/en/manual/api/reference/host/object](https://www.zabbix.com/documentation/current/en/manual/api/reference/host/object) 	
同时也提供了Inventory配置的代码Demo
[https://www.zabbix.com/documentation/current/en/manual/api/reference/host/update](https://www.zabbix.com/documentation/current/en/manual/api/reference/host/update)

Request:
```
{
    "jsonrpc": "2.0",
    "method": "host.update",
    "params": {
        "hostid": "10387",
        "inventory_mode": 0,
        "inventory": {
            "location": "Latvia, Riga"
        }
    },
    "auth": "038e1d7b1735c6a5436ee9eae095879e",
    "id": 1
}
```
Response:
```
{
    "jsonrpc": "2.0",
    "result": {
        "hostids": [
            "10387"
        ]
    },
    "id": 1
}
```
## 7.建议
1.导出功能：zabbix资产未提供导出功能，实际使用起来只能进行维护，不能导出，建议官方增加资产导出功能；
2.资产字段自定义：zabbix的资产字段目前只能展示特定的字段，不能实现字段的自定义，建议增加自定义显示字段，实现个性化显示；
### 8.ZbxTable 2.0版本功能预告
距离ZbxTable版本发布已经有2年多了，Zabbix 6.0 LTS也已经发布。ZbxTable 2.0版本将于近期发布，可适配最新的Zabbix 6.0版本，众多功能全新升级。
全新首页更直观，内存、CPU Top排名，设备归类统计
![6](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_210213.png)
简单资产树管理，代替人肉Excel管理，数据均保存与zabbix数据库，不使用独立数据库，对接zabbix原生资产管理支持Excel导出。
![7](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_210949.png)
设备总览，设备健康状态一目了然
![8](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_211628.png)
直观显示设备状态，可导出Excel
![9](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_212404.png)
系统详情了解机器实时运行情况
![10](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_212913.png)
网络设备列表，数据均来自zabbix自动采集
![11](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_213123.png)
网络设备详情页，可查看端口流量及状态
![12](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_213344.png)
物理服务器列表
![13](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_213417.png)
简单告警分析及告警TOP
![14](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_213625.png)
告警历史查询
![15](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_213835.png)
拓扑管理，可手动创建拓扑
![16](https://img.cactifans.com/wp-content/uploads/2022/04/2022-04-20_214322.png)
节点及连线数据绑定
![16](https://img.cactifans.com/wp-content/uploads/2022/04/20220420230419.png)
拓扑图效果
![16](https://img.cactifans.com/wp-content/uploads/2022/04/1111.gif)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
