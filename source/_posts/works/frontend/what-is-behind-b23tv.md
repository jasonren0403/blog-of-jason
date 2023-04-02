---
title: 谈点b23tv短链接相关
date: 2023-02-17 11:19:30
tags: ["frontend"]
categories:
  - ["frontend"]
  - ["技术"]
---

## 前言

日常网友聊天时，总是会从视频网站中分享点东西，在分享的众多网站中，又以[b站](https://www.bilibili.com)为主。在这个场景中，我想各位一定也会经常见到不少b站的短链接 `b23.tv` 域名了。这其中背后会有什么故事呢？

<!--more-->

顺便，我并没有参与过这方面的开发，只是以用户角度及可能的相关知识做一些分析。

## 短域名

### 由来

顾名思义，短域名一般长度很短，且不带www。这样的域名出现是为了方便记忆，网民输入网址的时候，不必要输入太长的域名。某些网站会申请一些短域名，然后将它们设置跳转到带www的顶级域名上。正因为长度足够短，在社交媒体上分享时，会节省很多字数。

### 实现
我们常用的情景有这两种：

1. 同域跳转，将不带www的域名直接转发到带www的域名，后台如果使用nginx，会配置成这样：

	```
	server {
		listen                  443 ssl http2;
		listen                  [::]:443 ssl http2;
		server_name             www.jason0743.ren;
	}

	server {
		listen      80;
		listen      [::]:80;
		server_name jason0743.ren;
	
		location / {
			return 301 https://www.jason0743.ren$request_uri;
		}
	}
	
	server {
		listen      80;
		listen      [::]:80;
		server_name www.jason0743.ren;
	
		location / {
			return 301 https://www.jason0743.ren$request_uri;
		}
	}
	```
	
	这个配置中，主要依靠 `location /` 块下指定的 301 状态码及跳转地址。这样，用户访问 `jason0743.ren` 时，会被重定向到 `www.jason0743.ren`，这样就可以统一使用带www的域名了。

	根据相关标准，代表永久重定向的 301 状态码会被浏览器缓存，搜索引擎也会将资源的索引转移到新地址上，所以说权重会聚集在带www的主域名上。

2. 非同域跳转。本文要介绍的 `b23.tv` 跳转到 `bilibili.com` 的属于这种情况。一般来说，二字母、二数字、二杂合的域名都是效果非常好的短链接的来源。这样的域名往往依靠注册得到。

	马上我们就会看到，它的实现方法也是类似于第一种情况，使用301、302状态码和跳转地址来实现。

短网址-长网址的对应关系，很容易想到一个数据结构——Map。这里我们可以使用一个Map来存储短网址和长网址的对应关系，然后根据短网址来查找长网址，再根据长网址来重定向。在实际应用中，这种对应关系往往使用数据库存储。每次生成短链接时，就将它插入数据库中，每插一次，就会关联一个id，下次可以通过id定位到原始链接，然后作后续跳转。

用数字作为id是很容易想到的，但是，短网址数据通常是上亿的，如果用户输入过长数字，体验想必也不会太好。所以，我们可以使用一些编码算法，将数字转换成更短的字符串。这里我们可以使用62进制，即0-9、a-z、A-Z，这样就可以用6位字符来表示62的6次方个数字，也就是568亿个数字。这样，我们就可以使用6位字符来表示一个id了。

短链接服务往往是高并发的，业务数据量非常大，不得不分库分表，分库分表的情况下，仍然需要唯一标识数据的ID，自增ID就不太好用了。这时，就需要一些分布式ID生成算法的帮助了，比如Redis的incr命令，Twitter的雪花算法等。这些东西要展开的话，可能需要额外一篇文章了。

## b站短域名到底有没有包含个人信息？

现在已经是2023年了，在bing等各大搜索引擎上也能找到b站添加短域名标识符的报告信息了。大家在看到这些信息时，肯定会说”又有应用收集我信息了，我隐私无了“吧！但是，你知道这些是什么信息吗？为了让大家明白，我们把访问b23短域名的过程来进行一个简单的测试。

从我们手机B站APP上，随便打开一个视频，然后点击右上角的三个点，选择“复制链接”。这时你会发现，剪贴板上的内容类似如下：

```
【视频标题】 https://b23.tv/{xxxxxx}
```

其中视频标题的右方括号与短链接之间有一个空格。我们将这个短链接拿去测试一下：

```python
import requests
# 别忘了把short-id换成另外一个合法链接自己测试
res = requests.get("https://b23.tv/<short-id>")
print(res.url)
# 'https://www.bilibili.com/video/BV{xxxxxxxxx}/?buvid={buvid}&is_story_h5=false&mid={mid}&p=1&plat_id={plat_id}&share_from=ugc&share_medium={medium}&share_plat={platform}&share_session_id={sid}&share_source=COPY&share_tag=s_i&timestamp={ts}&unique_k={}&up_id={}'
```

最后的 `url` 地址与初始请求地址不同，显然经过了30x的重定向，强制忽略重定向再来一遍：

```python
import requests
# 别忘了把short-id换成另外一个合法链接自己测试
res = requests.get("https://b23.tv/<short-id>", allow_redirects=False)
# <Response [302]>
print(res.headers)
# {'Date': 'Fri, 17 Feb 2023 03:55:30 GMT', 'Content-Type': 'text/html; charset=utf-8', 'Content-Length': '420', 'Connection': 'keep-alive', 'Bili-Trace-Id': '{trace-id}', 'Location': 'https://www.bilibili.com/video/BV{xxxxxxxxx}?{search-params}', 'X-Bili-Trace-Id': '{trace-id}', 'Expires': 'Fri, 17 Feb 2023 03:55:29 GMT', 'Cache-Control': 'no-cache', 'X-Cache-Webcdn': 'BYPASS from blzone06'}
```

`Location` 响应头的地址与不忽略重定向时的地址相同，这时候我们大致可以描绘出整个请求的过程了。

```mermaid
graph LR
  A["访问短链接b23.tv/xxx"] --> B["取Location进行302跳转"] --> C["带一堆URL参数的bilibili.com/BVxxxxxx视频页链接"]
```

{% note warning %}
看见没，已经有一堆追踪参数了，为了这里演示方便以及不泄露隐私，这里用占位符 `{}` 替换掉了所有可能引起追踪的地方。
{% endnote %}

可以看到在进行302跳转时，服务器给出的 `Location` 响应头已经携带了参数。这些参数中，有一些是与用户相关的，比如 
* `mid` 用户ID，在2023年4月2日测试时，mid已经变成了一堆字母和数字，不知道有没有作加密
* `buvid` 用户设备ID
* `plat_id` 用户平台ID
* `share_session_id` 分享会话ID，其格式为UUID
* `share_tag`，没猜到含义
* `share_from` 分享来源内容，ugc为用户生成内容（User Generated Content）
* `share_medium` 分享媒介，如安卓手机端的分享值为android
* `share_source` 分享来源，如安卓手机端的分享值为COPY
* `share_plat` 分享平台，如安卓手机端的分享值为android
* `up_id` UP主ID，也就是UP主空间主页 `space.bilibili.com` 后面那一串数字
* `timestamp` 时间戳，位数为10位

这些当然利于刻画分享视频用户的行为，便于后期综合到UP主的分析中。如[数据中心——播放](https://member.bilibili.com/platform/data-up/video/dataCenter/play)中，就使用到了用户的设备信息。

{% grouppicture 2-2 %}
{% asset_img datacenter-play-1.png "数据中心播放统计" %}
{% asset_img datacenter-play-2.png "数据中心播放统计" %}
{% endgrouppicture %}

而在[数据中心——观众](https://member.bilibili.com/platform/data-up/video/dataCenter/audience)中，互动活跃度统计会使用到分享的数据。

{% grouppicture 2-2 %}
{% asset_img datacenter-audience-1.png "数据中心观众统计" %}
{% asset_img datacenter-audience-2.png "数据中心观众统计" %}
{% endgrouppicture %}

既然分享也算一种互动，如果分享的人是粉丝，则它也会参与到粉丝粘性的计算中。

{% asset_img datacenter-audience-3.png "粉丝粘性" %}

在电脑端上获取视频分享链接，就直接是 `https://www.bilibili.com/video/BV{xxxxxxx}/?share_source=copy_web&vd_source={}` 这种长链接格式了。

## 总结

* 一些门户分享网站为了方便统计分析，在URL查询字段中会携带各种信息。无论有关无关，总会引起部分用户的不满，手动删除标识符总是一个好习惯。今后我也可能会开发一个工具来解除”标识符之苦“。
* 平衡用户体验和隐私保护，也是现在很多国内网站需要优化的地方。希望这些网站多多注意。
* 写这篇文章的意义，也是基于”不要人云亦云“的原则，对这件事有个自己的看法，并以文章的形式记录下来。希望这篇文章能够帮助到大家做进一步的判断，也希望B站短链接部门能够做好用户隐私和数据分析的平衡，有所改进吧。