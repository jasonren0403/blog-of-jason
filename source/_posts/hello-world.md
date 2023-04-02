---
title: Hexo用法备忘录 
comments: false
date: 2021/9/30 0:00:00
updated: 2023/1/1 0:00:00
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

<!--more-->

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### Tag Plugins Test

* Pullquotes

```
{% pullquote [class] %}
content
{% endpullquote %}
```

{% pullquote 2345 %} content {% endpullquote %}

* Other posts

```
{% post_path filename %}
{% post_link filename [title] [escape] %}
```

{% post_link works/frontend/echarts-drawing %}

* Admonition

```
{% note info %}
#### title
content
{% endnote %}

{% note warning title %}
content
{% endnote %}

{% note success %}
#### Success Header
**Welcome** to [Hexo!](https://hexo.io)
{% endnote %}
```

{% note info %}

#### title

content {% endnote %}

{% note warning title %} content {% endnote %}

{% note success %}

#### Success Header

**Welcome** to [Hexo!](https://hexo.io)
{% endnote %}

* Content Tabs

```
{% tabs Unique name, [index] %}
<!-- tab [Tab caption] [@icon] -->
Any content (support inline tags too).
<!-- endtab -->
{% endtabs %}
```

{% tabs Unique name,2 %}
<!-- tab first tab@heart -->
first
<!-- endtab -->
<!-- tab second tab-->
second
<!-- endtab -->
<!-- tab third tab-->
**This is Tab 3.**
<!-- endtab -->
{% endtabs %}

## Video and audio embeds

> Using `hexo-tag-mmedia` plugin, https://easyhexo.com/3-Plugins-use-and-config/3-5-hexo-tag-mmedia

### Bilibili

```text
{% mmedia "bilibili" "bvid:<bvid>" "quality:high" "danmaku" "allowfullscreen" %}
```

{% mmedia "bilibili" "bvid:BV1yN4y1N7vZ" "quality:high" "danmaku" "allowfullscreen" %}

### 西瓜视频
```text
{% mmedia "xigua" "xid=<xid>" %}
```

{% mmedia "xigua" "xid=7168109503483740711" %}

### MetingJS

```text
{% mmedia "meting" <id> <server> <type> ... %}
```

{% mmedia "meting" "id:7737308430" "server:netease" "type:playlist" %} 

