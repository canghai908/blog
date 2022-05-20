---
title: "setting up sms with cacti"
date: 2015-05-31 22:49:32
tags: [cacti, 短信，告警]
categories: [cacti]
---

之前听人说使用飞信告警，由于最近飞信机器人协议更新，不能使用，因此采用了第三方的短信接口，实现了 cacti 的短信告警。

# 接口申请

市面上短信接口很多，都提供了标准的 http 接口，我试用了[云片网络](http://www.yunpian.com/)的短信接口，申请好之后，要申请一个模版，不按照模版是不能发送的，另外还要添加你的主机 IP 地址到白名单里，不然不能发送成功，使用其他接口的请自行设置

模版配置

![模版配置](https://img.cactifans.com/wp-content/uploads/2015/05/13.png)

白名单设置

![IP白名单](https://img.cactifans.com/wp-content/uploads/2015/05/21.png)

## 接口测试

我们先放到一个文件到 cacti 服务器上，测试一下，使用 php 写了一个简单的测试发送页面，代码如下:

```php
<?php
/**
 * 在PHP 5.5.17 中测试通过。
 * 默认用通用接口(send)发送，若需使用模板接口(tpl_send),请自行将代码注释去掉。
 */

$subject = "haha"; //告警内容
//通用接口发送样例
$apikey = "AAAAAAAAAAAA"; //请用自己的apikey代替
$mobile = "13800121212"; //请用自己的手机号代替
$text="【云片网】告警内容:".$subject.",请及时处理故障！"; //模版
echo send_sms($apikey,$text,$mobile);


/**
* 通用接口发短信
* apikey 为云片分配的apikey
* text 为短信内容
* mobile 为接受短信的手机号
*/
function send_sms($apikey, $text, $mobile){
	$url="http://yunpian.com/v1/sms/send.json";
	$encoded_text = urlencode("$text");
	$post_string="apikey=$apikey&text=$encoded_text&mobile=$mobile";
	return sock_post($url, $post_string);
}

/*

* url 为服务的url地址
* query 为请求串
*/
function sock_post($url,$query){
	$data = "";
	$info=parse_url($url);
	$fp=fsockopen($info["host"],80,$errno,$errstr,30);
	if(!$fp){
		return $data;
	}
	$head="POST ".$info['path']." HTTP/1.0\r\n";
	$head.="Host: ".$info['host']."\r\n";
	$head.="Referer: http://".$info['host'].$info['path']."\r\n";
	$head.="Content-type: application/x-www-form-urlencoded\r\n";
	$head.="Content-Length: ".strlen(trim($query))."\r\n";
	$head.="\r\n";
	$head.=trim($query);
	$write=fputs($fp,$head);
	$header = "";
	while ($str = trim(fgets($fp,4096))) {
		$header.=$str;
	}
	while (!feof($fp)) {
		$data .= fgets($fp,4096);
	}
	return $data;
}

?>
```

也可下载我的测试脚本：[send.php](https://dl.cactifans.com/cacti/send.php)
文中的 apikey 和手机号，改成你自己的，保存到 cacti 的根目录下，保存为 send.php，用浏览器访问此文件，如果有错误会有提示，如果出现如下：
![返回](https://img.cactifans.com/wp-content/uploads/2015/05/31.png)

表示发送成功，同时手机也收到了短信，表示配置正确。

# cacti 配置

thold 是 cacti 的告警插件，可对主机等设备设置阈值，设置之后可通过 Email 等方式通知管理人员，这里我添加一个 Linux 用户登录数的告警，登录用户数大于 2 或者登录用户数小于 1，都会发短信通知指定的管理人员:
由于我们调用的事邮件函数，所以需要指定一个邮件通知列表，随便填就可以
邮件列表
![1](https://img.cactifans.com/wp-content/uploads/2015/05/111.png)
添加规则
![1](https://img.cactifans.com/wp-content/uploads/2015/05/41.png)
选择数据源
![2](https://img.cactifans.com/wp-content/uploads/2015/05/51.png)
规则
![3](https://img.cactifans.com/wp-content/uploads/2015/05/62.png)
添加好之后的效果
![4](https://img.cactifans.com/wp-content/uploads/2015/05/71.png)
看到 Triggered 为 no，表示还没有触发。
至此阈值配置完成，对于别的项目，可自行对照设置，基本大同小异

## 修改 Thold 插件

由于 Thold 插件没有自带输出到短信的接口，所以需要修改 thold 把告警的信息，通过短信接口发送出去,我是用的 thold 插件为最新的 0.5 版本
修改 thold 插件目录下的 thold_functions.php 文件（我的 cacti 安装目录为/data/wwwroot/cacti/）

```
vi /data/wwwroot/cacti/plugins/thold/thold_functions.php
```

文件 2722 行之后 添加如下内容：

```php
$apikey = "AAAAAAAAAAAAAAAAAAAA";
$mobile = "13800001234";
$text="【云片网】告警内容:".$subject.",请及时处理故障！";
echo send_sms($apikey,$text,$mobile);
```

添加好之后如图：
![6](https://img.cactifans.com/wp-content/uploads/2015/05/81.png)

文件 2855 行之后，添加二个发送函数，代码如下：

```php
/**
* 通用接口发短信
* apikey 为云片分配的apikey
* text 为短信内容
* mobile 为接受短信的手机号
*/
function send_sms($apikey, $text, $mobile){
	$url="http://yunpian.com/v1/sms/send.json";
	$encoded_text = urlencode("$text");
	$post_string="apikey=$apikey&text=$encoded_text&mobile=$mobile";
	return sock_post($url, $post_string);
}

/*

* url 为服务的url地址
* query 为请求串
*/
function sock_post($url,$query){
	$data = "";
	$info=parse_url($url);
	$fp=fsockopen($info["host"],80,$errno,$errstr,30);
	if(!$fp){
		return $data;
	}
	$head="POST ".$info['path']." HTTP/1.0\r\n";
	$head.="Host: ".$info['host']."\r\n";
	$head.="Referer: http://".$info['host'].$info['path']."\r\n";
	$head.="Content-type: application/x-www-form-urlencoded\r\n";
	$head.="Content-Length: ".strlen(trim($query))."\r\n";
	$head.="\r\n";
	$head.=trim($query);
	$write=fputs($fp,$head);
	$header = "";
	while ($str = trim(fgets($fp,4096))) {
		$header.=$str;
	}
	while (!feof($fp)) {
		$data .= fgets($fp,4096);
	}
	return $data;
}
```

添加好之后效果：
![7](https://img.cactifans.com/wp-content/uploads/2015/05/91.png)

也可直接下载我的[thold_function.php](https://dl.cactifans.com/cacti/thold_functions.php)覆盖你的，注意修改文件里的 apikey 和手机号码，注意备份你的原有文件。

# 最终效果

添加之后，我们登录四个终端，在 cacti 界面就会看到 thold 规则已经触发，手机会就收到短信。
触发
![11](https://img.cactifans.com/wp-content/uploads/2015/05/101.png)
手机收到短信，发送速度很快基本秒到
![12](https://img.cactifans.com/wp-content/uploads/2015/05/M1B1QQK05E7N9HRR1.png)

如有特殊需求，可发邮件到 cactifans#gmail.com，联系本人，提供功能定制服务。 （＃换成@）
