---
title: Cacti95计费配置
date: 2018-01-20 16:12:06
tags: [cacti]
categories: [cacti]
---

## 一.95 计费

95 计费（95th Percentile charging）95 计费是把一个结算时间里的流量（通常为一个月），按每 5 分钟统计一次，取流量最高值做一个点。这样一个月会得到很多流量峰值点。然后把图中高流量的 5%的点去掉，按照剩下（100-5）% 来计算费用。
如果是每月结一次款。每 5 分钟取一个流量最高点，1 个小时 12 个点，1 天 12×24 个点，一个月按 30 天算 12×24×30=8640 个点，然后把数值最高的 5%的点去掉，剩下的最高带宽就是 95 计费的计费值了。需要计费的点数是 8640-432=8208 个点。其中有 432 个点不用计费，就是异常高流量的时间为 432 点 ×5 分钟/60 分钟/小时=36 个小时，即每月不超过 36 小时的异常大带宽（流量），不影响本月的计费。

直接导入 95 计费模版，即可实现 95 计费
![5](https://img.cactifans.com/wp-content/uploads/2016/08/D620E97E-4AD3-4499-8BB8-DFD9FBF2854C.jpg)

## 二.95 计费聚合

95 聚合可聚合几个端口的流量，端口不需要在一个主机，可以分布在多个主机
创建图形，这里选择图形在那个主机下查看
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-08_104120.png)
默认不用改，修改图形名称
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-08_104915.png)
添加聚合端口
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-08_105039.png)
选择需要聚合的主机端口，注意类型
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_230545.png)
如图，依次添加需要聚合的端口
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_230707.png)
添加总流量数据
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_231223.png)
总流量
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_231301.png)
添加聚合 Current 流量
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_231447.png)
添加聚合 Average 流量
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_231719.png)
添加聚合 Max 流量
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232042.png)
添加累计流量图
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232206.png)
效果
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232224.png)
添加 95 线
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232348.png)
效果
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232405.png)
添加聚合 95 流量数字
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232520.png)
最终效果
![2](https://img.cactifans.com/wp-content/uploads/2016/08/2016-08-14_232540.png)
关于 95 计费的具体参数，可以查看 cacti 官方文档
[cacti 参数](http://www.cacti.net/downloads/docs/html/variables.html#GRAPH_VARIABLES)

**如果觉得我的文章对您有用，请关注我的公众号，有更多技术干货！**
![微信](https://img.cactifans.com/wp-content/uploads/2017/12/qrcode_for_gh_5c46969f2957_258-1-1.jpg)
