---
title: 给自己的小破站上个HTTPS
tags:
  - 运维
  - 建站
categories:
  - 运维
  - 技术
excerpt: 本文记录了个人网站配置HTTPS的过程，包括github pages、certbot等工具的配置过程。
date: 2023-04-21 20:12:00
---


## 为何配置 HTTPS

HTTPS 是 HTTP 的安全版，使用了 SSL/TLS 协议对通信进行加密，防止被窃听、篡改、伪造等，是现代网站必备的安全配置。现代的浏览器也偏向对那些没有使用 HTTPS 的网站标记为”不安全“，且 HTTPS 对于用户和网站开发者来说，都可以提供安全的保证，即使并不一定涉及到敏感数据的传输，也可以防止 ISP 注入广告，引起普通用户的反感。对于普通用户来说，是值得信赖的认证。

现如今，HTTPS 技术经过不断发展，已经变得相当方便，成本也越来越低，即使是个人用户，也可以至少申请一份一年的免费证书，通过一些配置安装到自己的网站上（虽然额度有限，当然一些云服务商肯定会让你去买正式证书）。

写这篇文章的时候（2023.5），我去看了一下[腾讯云SSL证书购买页面](https://buy.cloud.tencent.com/ssl)，目前的定价对于个人用户来说，还是白嫖好了：（打折后/打折前，单位为元）

|         |   单域名证书    |   泛域名证书    |      多域名证书      |
|:-------:|:----------:|:----------:|:---------------:|
| 个人（DV型） |  532/560   | 2147/2260  |  912/960(3个域名)  |
| 企业（OV型） | 2470/2600  | 5662/5960  | 5282/5560(3个域名) |


## GitHub Pages 启用 HTTPS

有几个域名是使用 GitHub Pages 托管的，其开启 HTTPS 的方式很容易。

1. 首先进入要配置的仓库，点击上方的 `Settings` 进入配置。
2. 点击左侧的 `Pages`，在 `Build and deployment` 中选择要部署的分支，点击 `Save` 保存。
3. 在 `Custom domain` 部分填写自己的域名（如果有），点击 `Save` 保存。GitHub 会立即进行一次检查。

	{% gp 2-2 %}
	{% asset_img github-pages-1.png %}
	{% asset_img github-pages-2.png %}
	{% endgp %}

	{% note %}
	如果使用的是github.io默认域名，则 HTTPS 会自动开启，无需手动配置。
   	{% endnote %}

4. 一开始，这里一定是检查失败的，因为还没有按照GitHub的要求正确配置完毕。

	{% asset_img github-pages-3.png %}

5. 我的域名是在 DNSPod 注册的，现在转到 DNSPod 控制台，添加一个 CNAME 记录，其主机记录填写自定义域名的主机部分，记录值填写 `<你的github用户名>.github.io`，TTL 保持默认，保存。

	{% asset_img github-pages-4.png %}

6. 回到刚才的Pages设置页面，DNS检查已经通过了，现在GitHub已经开始申请TLS证书了。

	{% asset_img github-pages-5.png %}

7. 等待一段时间，证书申请完毕，就可以把下方的 `Enforce HTTPS` 勾选上了，这样站点就可以仅通过HTTPS访问了，更加安全。

	{% asset_img github-pages-6.png %}

{% note %}
可以在项目设置中勾选“Use your GitHub Pages website”，省去每次想不起域名的麻烦，也可以更方便地访问部署所在网页。
{% asset_img gh-setting.png %}
{% endnote %}

## 不怕麻烦的nginx手动配置过程

{% note %}
不要忘记在安全组设置中放开443端口哦~
{% endnote %}

### 申请证书

腾讯云对于每个个人用户都有50张免费证书的额度，对于我这种轻度开发用户来说，当然是一个域名一张证书就够用了，而且所需要的证书类型也是最简单的DV型，所以申请证书的过程也很简单。

{% asset_img tcloud-1.png "呃，有了之后要介绍的技术这些打叉的项算什么嘛" %}

根据[文档](https://cloud.tencent.com/document/product/400/6814)介绍和申请页所示，申请的证书是 TrustAsia 颁发的一年期证书，且同一域下最多有20张免费证书。这一步完成后，会获得颁发的证书，接下来需要将证书部署到相关服务上。

### 部署证书

首先当然要从[控制台](https://console.cloud.tencent.com/ssl)中下载证书。

nginx 中，SSL 的配置项主要在配置文件的 `server` 块中，为了支持 HTTPS，至少需要添加如下配置，其中 `DOMAIN` 是你申请证书的域名：

```nginx
server {
     listen 443 ssl; 
     server_name <DOMAIN>; 
     ssl_certificate /path/to/<DOMAIN>_bundle.crt; 
     ssl_certificate_key /path/to/<DOMAIN>.key; 
     ssl_session_timeout 5m;
     ssl_protocols TLSv1.2 TLSv1.3; 
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
     ssl_prefer_server_ciphers on;
     # ... location 配置省略
 }
```

将证书使用 WinSCP 等工具上传至服务器，然后在配置文件中填写正确的路径即可。

如果需要将 HTTP 请求重定向到 HTTPS 中，需要对配置的主机增加 80 端口监听，并将其转到 HTTPS 版本的链接上：

```nginx
server {
	listen 80;
	server_name <DOMAIN>;
	return 301 https://$server_name$request_uri;  // 这行是关键
}
```

## certbot 申请证书大作战

之前一段时间一直在使用 certbot 的 dnspod 脚本来自动申请和续期证书，但是最近发现这个脚本突然不可用了，故趁这个机会总结一下手动申请证书的过程。

我的服务器操作系统是ubuntu 18，使用nginx承担网站的日常工作。

{% note %}
[certbot 官方网站](https://certbot.eff.org/instructions)上分系统地提供了安装的方法，具体可以跟随那里操作。
{% endnote %}

### 先安装

1. certbot需要使用snapd安装，如果没有安装的话，需要使用以下命令安装：

	```bash
	sudo apt update
	sudo apt install snapd
	```

2. 保证snapd为最新版本

	```bash
	sudo snap install core
    sudo snap refresh core
	```

3. 卸载目前系统中残留的certbot组件

	```bash
	sudo apt-get remove certbot
	```

4. 安装certbot

	```bash
	sudo snap install --classic certbot
	```

5. 创建软链接

	```bash
	sudo ln -s /snap/bin/certbot /usr/bin/certbot
	```

6. 验证安装

	```bash
 	sudo certbot --version
    ```

### 撤销之前错误的证书

需要使用 `sudo certbot certificates` 查看当前已经申请的证书，如果有错误的证书，可以使用 `sudo certbot revoke --cert-path /etc/letsencrypt/live/xxx/fullchain.pem` 撤销。

### nginx 申请模式

1. 直接运行 `sudo certbot --nginx`，certbot 会自动扫描所有有效的 nginx 配置文件，列出所有可以申请证书的域名，选择需要申请的域名，输入序号，回车。或者，使用 `-d` 参数指定需要申请的域名，例如 `sudo certbot --nginx -d example.com`。

	{% asset_img certbot-1.png %}

	{% note %}
	如果指定多次 `-d` 参数，则可以进行多域名申请。
	{% endnote %}

2. 等待脚本申请完成。

	{% note warning no-icon %}
	{% asset_img certbot-3.png %}
	这种情况通常是之前配置过，没有清理干净，把报错提到的那行改成这个就行：
	
	```nginx
	return 301 https://$host$request_uri;
	```
	
	然后重新运行
	
	```bash 
	sudo certbot install --cert-name xxxx
	```

	即可继续配置
	{% endnote %}

3. 然后可以使用 `sudo certbot certificates` 看到自己刚才申请的证书了，现在有90天有效期。

	{% asset_img certbot-2.png %}

	{% note success %}
    有些博客会引导我们手动去配cronjob任务，但实际上，在刚才的部署过程中，certbot会自动将更新命令加入到系统的cronjob中，在临期时自动续期证书。其位于 `/etc/cron.d/certbot` 中，源码如下

	```bash
    0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew
    ```
	{% endnote %}

4. 使用 `sudo certbot renew --dry-run` 测试一下续期命令是否可用。以后，可以使用不带 `--dry-run` 的命令来手动续期。
5. 访问网站，发现已经可以使用HTTPS访问了。

	{% asset_img certbot-4.png %}

### 泛域名证书申请

> 泛域名证书是指可以同时为多个子域名申请证书的证书，例如 `*.example.com` 可以为 `a.example.com`、`b.example.com` 等多个域名申请证书。

上述的方法是针对单个域名的，虽然一次可以申请多个域名，但那些域名都是预先有配置的，如果以后想再添加的话需要手动加入配置，非常麻烦。

泛域名的证书需要DNS解析服务商支持自动验证，其需要通过官方插件来实现自动更新DNS以通过监测，很可惜，截至目前，并没有dnspod的插件，所以只能手动更新DNS了。

{% note info %}
在安装certbot插件（`sudo snap install certbot-dns-<PLUGIN>`）前，需要先运行如下命令

```bash 
sudo snap set certbot trust-plugin-with-root=ok
```

以保证其具有与certbot snap同样的classic containment。
{% endnote %}

以下讲述手动更改配置的过程。

#### 手工通过验证

首先删除之前的证书

{% note warning %}
过期的证书不能撤销 `revoke`，只能删除 `delete`
{% endnote %}

```bash
sudo certbot delete --cert-name xxxx
```

然后运行该命令

```bash 
sudo certbot certonly --preferred-challenges dns --manual -d *.<要申请的域名> --server https://acme-v02.api.letsencrypt.org/directory
```

{% note warning %}
注意必须在 `--server` 中指定 `https://acme-v02.api.letsencrypt.org/directory` ，Let's Encrypt V2版本才支持泛域名解析。
{% endnote %}

然后会提示你添加一个TXT记录，访问你的DNS服务商后台，将给定的TXT记录添加，添加后等待一段时间，然后回车，就可以申请成功了。

{% gp 2-2 %}
{% asset_img certbot-wildcard-1.png %}
{% asset_img certbot-wildcard-2.png %}
{% endgp %}

如果配置正确，certbot 会显示部署成功

{% asset_img certbot-wildcard-3.png %}

{% note warning %}
正如上个截图所说的那样，这种认证方式只有90天的有效期，且不能使用 `certbot renew` 的方式更新证书，所以需要手动续期，续期的时候，需要再次添加TXT记录。对于没有API支持的服务商来说，也只能这么做了。
{% endnote %}

完成以后，需要手动向配置文件增加相关配置。如果使用的是 nginx，则在站点的 nginx 配置文件中，将证书换为 Let's Encrypt 提供的证书即可：

```nginx
server{
    listen 443 ssl;
    # ...
    ssl_certificate /etc/letsencrypt/live/<要申请的域名>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<要申请的域名>/privkey.pem;
}
```

过几分钟后查看，就可以看到已经成功申请了泛域名证书。

{% asset_img certbot-wildcard-4.png "注意CN字段部分" %}
