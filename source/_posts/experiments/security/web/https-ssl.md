---
title: HTTPS与SSL实验 date: 2019-12-10 19:42:33 tags:

- "网络空间安全"
- "网络安全"
  categories:
- ["实验","网络安全"]

---

{% note warning %} 本篇文章未整理成正式实验记录，故可能出现书写或表述不完整的情况。 {% endnote %}

<!-- more -->
{% note danger %} 本篇文章为实验记录，仅供交流学习使用，切勿违法应用，所有文中提到的工具不提供下载。 {% endnote %}

## OpenSSL编译及代码书写

有点小坑，需要使用安装版vc6.0，且需要在生成do_ms后将ntdll.mak中CFlags中的`/WX`选项去掉。运行`nmake -f ms/ntdll.mak`生成的是动态库。 {% note info %}
不需要ipv6库，所以在configure时要指定`-DOPENSSL_USE_IPV6=0`
{% endnote %}

{% asset_img 2.png "Configure命令" %} {% asset_img 1.png "perl 版本" %} {% asset_img 11.png "测试成功" %}

为方便使用，将bin的路径加入系统变量path中。这样可以在命令行中方便地启动`openssl`
{% asset_img 12.png "加入path" %} {% asset_img 13.png "命令行启动openssl工具" %}

### 配置OpenSSL测试环境

1. 在`openssl\openssl.cnf`下设置证书缺省字段 {% asset_img 14.png "默认签名用信息" %}
2. 生成客户端和服务端的证书请求文件和私钥 {% codeblock lang:shell line_number:false %} openssl req -newkey rsa:1024 -out req1.pem -keyout
   sslclientkey.pem openssl req -newkey rsa:1024 -out req2.pem -keyout sslserverkey.pem {% endcodeblock %} {% asset_img
   15.png "签署新的CA" %} {% asset_img 16.png "证书信息" %} {% asset_img 17.png "生成私钥" %} {% asset_img 18.png "生成私钥2" %}
3. 执行命令，签发客户端和服务端证书。 {% codeblock lang:shell line_number:false %} openssl ca -in req1.pem -out sslclientcert.pem openssl
   ca -in req2.pem -out sslservercert.pem {% endcodeblock %} {% asset_img 19.png "签发证书1" %} {% asset_img 20.png "签发证书2"
   %}
4. 执行命令，运行ssl服务端和客户端 {% codeblock lang:shell line_number:false %} openssl s_server -cert sslservercert.pem -key
   sslserverkey.pem -CAfile demoCA/cacert.pem -ssl3 openssl s_client -ssl3 -CAfile demoCA/cacert.pem {% endcodeblock %}
   {% asset_img 23.png %} {% asset_img 24-1.png %} {% asset_img 24-2.png %}

### OpenSSL编程

1. 源代码 {% gist c6e99430d8f6d7f3ee82b971590ff9c0 sha1.cpp %}

2. 与标准CRC SHA工具的运算结果比较，完全相同，说明算法正确。 {% tabs opensslcode, 1 %}

<!-- tab 代码运行 -->
{% asset_img 21.png %}
<!-- endtab -->
<!-- tab 标准工具 -->
{% asset_img 22.png %}
<!-- endtab -->
{% endtabs %}

## HTTPS网站抓包及过程简析

{% note default %} 为了好截图以呈现结果，找了一个内容比较少的小站 https://www.fantasyroom.cn (也是我的小站咯~)
{% asset_img 3.png "浏览器上的小锁图标也可意味着其受到SSL的保护" %} IP地址为49.235.250.44（现在已更换，所以不作打码处理咯~） {% endnote %}

1. 可以看到几个TLSv1.2标志的包 {% asset_img 4.png %}
2. 首先，客户端发送client hello，协商密钥参数等信息 {% asset_img 5.png %} {% asset_img 6.png %}
3. 服务器端发回server hello，这个server hello包含认证信息和服务端的密钥交换。这次交换用到了椭圆曲线DH协议。 {% asset_img 7.png %}
4. 客户端的密钥交换。握手消息此时已经处于加密状态。 {% asset_img 8.png %}
5. 双方以加密方式互相通信应用数据。需要新的session时，由server端发起change cipher spec请求。 {% asset_img 9.png %} {% asset_img 10.png %}
