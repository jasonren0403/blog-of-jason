---
title: 我是如何一步步把这个博客搭建起来的
tags:
  - 运维
  - 建站
categories:
  - - 运维
  - - 技术
draft: true
date: 2023-04-03 19:45:33
---


好久不见！经过许久的折腾以后，终于闲下功夫来复盘一下博客如何部署起来的了！话不多说，这就进入主题！

<!--more-->

## 选择博客主题和架构

作为一个主要承担内容输出的部分，我首先在自行开发和成熟博客框架上进行了趋于中间化的选择。成熟的框架，如WordPress，固然可以完整提供博客所具有的全部功能，但是略显笨重，且对于本来空间就不大的云服务器更是一种考验。如果全部自己撰写的话，又会陷入到维护成本过高的问题。各种小部件的适配和样式的维护，都是一件很痛苦的事情。由此看来，静态博客生成器是一个不错的选择。

GitHub上有很多成熟的静态博客生成器，如Hexo、Jekyll、Hugo等等。进行了一番对比后，我最终选择了[Hexo](https://hexo.io) 和[Hexo-next](https://theme-next.js.org/) 主题。Hexo本体是基于Node.js的，我可以专心使用Markdown书写内容，而由一些hexo终端命令来渲染成所需的静态文档。而且，经过对一些GitHub仓库的浏览，我发现Hexo的主题和插件也是非常丰富的，可以满足我对于博客的基本需求。

## 主题安装与配置

Hexo的文档很清楚，我可以直接通过npm命令来安装Hexo本体，并安装Hexo next主题：

```bash
npm install hexo-cli -g
npm install hexo-theme-next
```

安装完成后，博客根目录下会出现两种yml配置文件，一个是`_config.yml`，称为Hexo配置文件；一个是`_config.next.yml`，称为主题配置文件。接下来就需要对这两个文件进行深一步配置。

### Hexo配置文件的进一步配置

首先是基本信息的填写，如博客标题、副标题、描述、关键词、作者、语言、时区等等。所有这些配置都将写入HTML的`meta`标签中，以便搜索引擎更好的索引博客。

```yaml
title: '随心记'  # 标题
subtitle: '展现安全态度'  # 副标题
# 浏览器标签页显示的标题会是 title - subtitle 的形式
description: ''  # 站点介绍
keywords: ''  # 关键词
author: ''  # 作者
language: zh-CN  # 所用语言，中国大陆为zh-CN
timezone: 'Asia/Shanghai'  # 时区，中国大陆网站最好设置为上海时区
```

我们安装的主题是hexo next主题，要在主配置文件（`_config.yml`）中写入主题的名称：

```yaml
theme: next
```

以上步骤结束，我们就可以通过 `hexo s` 命令来启动本地服务器，通过 `localhost:4000` 来预览博客的样子了。但是，我们还需要对主题配置文件进行进一步配置。

### next主题配置文件的进一步配置

{% note info %}
官方配置文档在[这里](https://theme-next.js.org/docs/theme-settings/)，可供参考
{% endnote %}

我所安装的Hexo已经是5.0版本以后了，官方推荐使用替代配置文件`_config.next.yml`来进行主题配置，以防止更新时被覆盖。

首先是博客主题（scheme），NexT提供了4种主题（`Muse`、`Mist`、`Pisces`和`Gemini`），我直接选择了默认主题 `Muse`，可能我个人对这个也没啥大要求吧，哪天可以改改试试其他的。

```yaml
scheme: Muse
```

然后是暗黑模式，这可以通过`darkmode`来设置。我并没有改变这个选项，感觉这个主题应该还是提供一个按钮，供用户切换比较好。

```yaml 
darkmode: true
```

接下来是站点Favicon，是浏览器标签页上会显示的小图标，需要在`favicon`下进行设置，并引用相关图片资源文件。目前本站还没有进行这样的设置，以后会加上。

```yaml 
favicon:
  small: /images/favicon-16x16-next.png
  medium: /images/favicon-32x32-next.png
  apple_touch_icon: /images/apple-touch-icon-next.png
  safari_pinned_tab: /images/logo.svg
```

有关博客文章的版权声明，鉴于本博客大多以分享为主，我沿用了主题提供的CC4.0配置，意味着放开了共享和演绎（虽然我的文章一般没啥人看就是了qwq）。这个选项的配置效果决定了文章底部的版权声明显示。

```yaml 
creative_commons:
  license: by-nc-sa
  sidebar: false
  post: true
  language:
```

为了优化SEO，OpenGraph当然是必须配置的。这里参考[hexo的配置文档](https://hexo.io/docs/helpers#open-graph)配置了一些项目。

```yaml 
open_graph:
  enable: true
  options:
  # 配置twitter_id、twitter_profile等
```

目录设置需要在`menu`选项下进行配置，其配置分为`Key`、`link`和`icon`三部分。其中：

* `Key`是目录项的名称，大小写敏感。除了`home`和`archive`以外，其他类型都需要在`source`文件夹下手动创建文件夹和对应的`index.md`文件
* `link`是目录项的链接，是链接到本站的相对路径，在实际测试时发现也可以写站外的绝对路径
* `icon`是目录项的Font Awesome图标名，可以在[这里](https://fontawesome.com/)找到
	- `link`和`icon`之间用`||`分隔

```yaml
menu:
  home: / || fa fa-home
  about: https://www.jason0743.space/#/about || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
  sitemap: /sitemap.xml || fa fa-sitemap
```

{% asset_img menu.png "显示效果" %}

`menu` 下还可以配置 `sub-menu`，其配置和 `menu` 类似，但是本站没必要配置这个，就不写了。

另外，在 `menu_settings` 还可以配置使用 `badges` 或者 `icons`。`badges: true` 会显示文章、类别和标签的数量。默认的 `icons: true` 会在 `menu` 项前显示图标。

#### 侧边栏

博客中难免需要显示一些个人相关的社交平台联系链接。这些链接可以通过 `social` 配置项进行配置。它的格式与上述 menu 类似，但是指定的链接必须是绝对链接。

```yaml 
social:
  GitHub: https://github.com/jasonren0403 || fab fa-github
  Twitter: https://twitter.com/theRealRev270 || fab fa-twitter
  Facebook: https://www.facebook.com/renjason1999/ || fab fa-facebook
  Instagram: https://www.instagram.com/jasonren0403/ || fab fa-instagram
```

通过 `social_icons` 配置项可以进行进一步配置，如是否启用（`enabled`）、是否只显示图标不显示描述（`icons_only`）、是否显示过渡动画（`transition`）等。

中国常见的社交平台对应的 Fontawesome 图标名称总结如下：

| 社交平台名称 | Fontawesome 图标名称 |
|:------:|:----------------:|
|   知乎   |   fab fa-zhihu   |
|   微博   |   fab fa-weibo   |
|   微信   |  fab fa-weixin   |
| QQ  	  |    fab fa-qq     |
|   抖音   |  fab fa-douyin   |
|   人人   |  fab fa-renren   |
|   B站   | fab fa-bilibili  |

对于单个博客文章，`Table of Contents(TOC)` 是非常必要的，它可以显示你的文章的大致结构。可以通过 `toc` 配置项进行启用和禁用。我的配置如下：

```yaml
toc:
  enable: true
  number: true
  wrap: true
  expand_all: true
  max_depth: 6
```

侧边栏所展现的个人头像可以通过 `avatar` 来进行配置，NexT 主题还支持使用 `rounded` 属性设置圆形头像，`rotated` 属性支持鼠标悬停时旋转头像。

{% asset_img sidemenu.png "侧边栏头像效果" %}

#### 页脚

以下配置需要在 `footer` 选项下进行配置。

##### 备案信息

众所周知，开在中国大陆的站点都需要备案，而备案信息是需要在页脚显示的。这里可以通过 `beian` 配置项进行配置，如下：

```yaml
beian:
  enable: true
  icp: 京ICP备xxxxxxxx
  gongan_id:
  gongan_num:
  gongan_icon_url:
```

##### 建站时间

一般建站的时候，总会往页脚加入开站时间，类似于 `2020-2023` 这样的。默认情况下，NexT 主题会显示当前年份，可以通过 `since` 配置项手动显示建站起始时间。

```yaml
footer:
  since: 2020  # 年份，一般表示建站时间
```

##### 页脚图标

页脚图标会显示在年份和版权信息之间，往往是一个红心图标。可以通过 `icon` 选项做更多自定义配置，如修改图标名称、修改图标颜色、使图标动起来等。

##### 版权信息

版权信息显示在页脚图标右侧，由 `copyright` 配置中的内容指定，如果未指定，则使用 Hexo 配置文件中的 `author` 配置项。

##### 生成平台信息

在默认情况下，NexT 主题会在页脚显示 `Powered by Hexo & Next.xxx`，表示生成器和主题，可以通过 `powered` 配置项进行更改。我觉得留着也没什么问题，就没改。

#### 文章

### 配置域名

根据[GitHub相关配置文档](https://docs.github.com/zh/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)和[Hexo文档](https://hexo.io/zh-cn/docs/github-pages.html)，需要在项目根目录下创建一个`CNAME`文件，里面写入你的域名，如`example.com`。由于hexo部署时相当于把整个source文件夹复制到一个新的分支，故这个`CNAME`文件需要创建在`source/`文件夹下[^1]。

{% asset_img cname.png 项目结构及CNAME文件 %}

## 安装插件

### 评论显示

静态站并没有“后端”这个概念，传统的评论系统该怎么实现呢？我们可以借助一些插件来实现。

我在这里的选择是 [Waline](https://waline.js.org/)，比起之前的Valine来说，更加安全[^2]，但配置起来需要一定时间。它需要同时考虑到server和client的部署问题。

{% note info %}
Waline 官方提供了[部署文档](https://waline.js.org/guide/get-started/)，所有步骤跟随这个走就行。
{% endnote %}

#### 服务端配置

{% note info %}
LeanCloud注册当然要选[国际版](https://leancloud.app/)，懂得都懂~
{% endnote %}

首先是LeanCloud端数据库的配置，注册账号和创建应用的过程就不赘述了（作为普通白嫖用户当然要选择开发版），然后到左侧的设置→应用凭证中，记下AppID、AppKey和MasterKey，以后要用到。

{% asset_img leancloud-1.png %}

接下来使用[Vercel](https://vercel.com/)进行服务端部署，如果没有账户的话，可以直接使用GitHub账号登录。然后点击Waline文档中的Vercel部署按钮或[直接访问这个仓库](https://github.com/walinejs/waline/tree/main/example)的README.md文件。

接下来会基于一个简单的模板创建Git仓库，作为 GitHub 常驻用户肯定时要选择GitHub的，如果有别的平台也可以点击别的平台按钮创建。

{% asset_img vercel-1.png "选择仓库" %}

项目创建成功后，进入到控制台中，从上方的“Settings”中进入项目设置，然后点击左侧的“Environment Variables”，进入环境变量配置，建立三个环境变量 `LEAN_ID`、`LEAN_KEY`、`LEAN_MASTER_KEY`，分别对应上一步在LeanCloud获得的AppID、AppKey和MasterKey。

{% grouppicture 2-2 %}
{% asset_img vercel-2.png "环境变量配置1" %}
{% asset_img vercel-3.png "环境变量配置2" %}
{% endgrouppicture %}

然后点击顶部的“Deployments”进入部署页面，选择最近一次部署，点击“Redeploy“使得刚才的环境变量生效。部署好后就可以回到主页点击”Visit“访问服务端了。

{% asset_img vercel-4.png "回到主页" %}

其界面大致是这样的

{% asset_img vercel-5.png "部署完毕" %}

如果需要加入域名（vercel默认域名实在有些丑），则依然需要进入设置页中，这次需要进入”Domains”中，输入需要绑定的域名并点击“Add”添加域名，然后在服务商处加入CNAME解析记录，其值为 `cname.vercel-dns.com`，之后就可以用自己的域名来访问评论系统和管理端了。

{% asset_img vercel-6.png %}

立即访问管理端（`<url>/ui/register`）进行注册，注册成功后，以后就可以通过 `<url>/ui` 管理评论了。

{% note info %}
这里最好通过账号绑定设置（`<url>/ui/profile`）到自己常用账户上，防止忘记登录凭证。
{% endnote %}

{% note info %}
`<url>` 为vercel域名设置中所列出的域名，项目新建的默认值一般为 `<project-name>.vercel.app`
{% endnote %}

{% asset_img management.png "管理界面" %}

#### 客户端配置

客户端需要进行样式导入和代码注入，对于hexo-next来说，像下面这样安装[插件](https://www.npmjs.com/package/@waline/hexo-next)并加以配置即可。

```bash 
npm install @waline/hexo-next
```

然后在主题配置文件 `_config.next.yml` 中加入以下内容：

```yaml
waline:
  enable: true
  serverURL: 
  libUrl:
  cssUrl:
  locale:
    placeholder: 
  commentCount: true
  pageview: true
  emoji:
    - https://unpkg.com/@waline/emojis@1.0.1/weibo
    - https://unpkg.com/@waline/emojis@1.0.1/alus
    - https://unpkg.com/@waline/emojis@1.0.1/bilibili
    - https://unpkg.com/@waline/emojis@1.0.1/qq
    - https://unpkg.com/@waline/emojis@1.0.1/tieba
    - https://unpkg.com/@waline/emojis@1.0.1/tw-emoji
  meta:
    - nick
    - mail
    - link
  requiredMeta:
    - nick
  lang: zh-CN	
  wordLimit: 0
  login: enable
  pageSize: 10
```

首先当然要启用该插件（`enable:true`）。然后需要设置 `serverURL` 为刚才部署的服务端地址，如果使用了自定义域名的话，这里就需要填写自定义域名。语言当然设置为 `zh-CN`，我也启用了 `pageview` 和 `commentCount` 两个功能，显示文章的阅读统计和评论数量，配置效果如下：

{% asset_img comment.png "评论区效果" %}

### 视频/音乐嵌入

博客如果只有静态的文字和图片的话，也未免太单调了吧！如果能适时地嵌入一些其他网站或自己上传的资源，就更好了。我选择的是 [mmedia](https://www.u2sb.com/OpenSw/hexo-tag-mmedia/install.html) 插件，因为它目前支持较广，而且使用也比较方便。

针对hexo，只需要安装 [hexo-tag-mmedia](https://www.npmjs.com/package/hexo-tag-mmedia) 插件即可，安装方法如下：

```bash
npm install hexo-tag-mmedia --save
```

然后在 `_config.yml` 下采用作者推荐的默认设置即可：

```yaml
mmedia:
  audio:
    default:
  video:
    default:
  aplayer:
    js: https://cdn.jsdelivr.net/npm/aplayer@1/dist/APlayer.min.js
    css: https://cdn.jsdelivr.net/npm/aplayer@1/dist/APlayer.min.css
    default:
      contents:
  meting:
    js: https://cdn.jsdelivr.net/npm/meting@2/dist/Meting.min.js
    api:
    default:
  dplayer:
    js: https://cdn.jsdelivr.net/npm/dplayer@1/dist/DPlayer.min.js
    hls_js: https://cdn.jsdelivr.net/npm/hls.js/dist/hls.min.js
    dash_js: https://cdn.jsdelivr.net/npm/dashjs/dist/dash.all.min.js
    shaka_dash_js: https://cdn.jsdelivr.net/npm/shaka-player/dist/shaka-player.compiled.js
    flv_js: https://cdn.jsdelivr.net/npm/flv.js/dist/flv.min.js
    webtorrent_js: https://cdn.jsdelivr.net/npm/webtorrent/webtorrent.min.js
    default:
      contents:
  bilibili:
    default:
      page: 1
      danmaku: true
      allowfullscreen: allowfullscreen
      sandbox: allow-top-navigation allow-same-origin allow-forms allow-scripts allow-popups
      width: 100%
      max_width: 850px
      margin: auto
  xigua:
    default:
      autoplay: false
      startTime: 0
      allowfullscreen: allowfullscreen
      sandbox: allow-top-navigation allow-same-origin allow-forms allow-scripts allow-popups
      width: 100%
      max_width: 850px
      margin: auto
```

在使用上，假如要嵌入一个B站的视频，只需要使用这个简单语法：

```markdown
{% mmedia "bilibili" "bvid:BV<xxxxx>" %}
```

即可，其中第一个参数是要引用的视频网站名称，第二个参数是要引用的视频id，以冒号或者等号分隔都可以。

如果要嵌入音乐播放列表，则需要采用其中的MetingJS标签形式，采用的参数可以在[这里](https://github.com/metowolf/MetingJS)了解。

```markdown
{% mmedia "meting" "server=<netease|tencent|kugou|xiami|baidu>" "type=<song|playlist|album|search|artist>" %}
```

其支持的平台其实已经能够涵盖常见的用途了。如果以上几种不够用，还有最后的 Audio 和 Video 参数打底，它可以支持上传自有资源，插入原生的标签。

{% note warning %}
不过[这个插件的原作者说他要弃坑了](https://github.com/u2sb/hexo-tag-mmedia/issues/23)，保证稳定性的话，还是最好找个替代品，或者自己二次封装一下比较合适。
{% endnote %}

## 部署到GitHub

由于将博客放到云服务器上操作过程过于繁琐（万一域名失效了还得四处拷贝搬运，并且配置域名HTTPS啥的也是不少麻烦），我选择使用 GitHub Pages 功能来托管我的内容。GitHub Pages 是 GitHub 提供的一个静态网页托管服务，可以将静态网页托管到 GitHub 上，然后通过 `username.github.io` 的形式来访问。这样做的好处是，我可以将博客的源码和静态网页内容分开，而且可以通过 GitHub Actions 来实现自动化部署。

### 建立仓库

在GitHub上建立一个新的仓库，并做好初始化，稍后我们会将博客的源码推送到这个仓库上。

本地的博客文件夹中，现在应该已经是这样的结构：

```
+--+
   |---scaffolds
        |---draft.md
        |---page.md
        |---post.md
   |---source
        |---_drafts
        |
        |---_posts
        |
        |---categories
             |---index.md
        |---tags
             |---index.md
        |---CNAME
   |---_config.next.yml
   |---_config.yml
   |---package.json
   |---package-lock.json
   |....
```

在这个文件夹下运行 `git init`，之后再把这个分支推送到仓库中。

目前，我的博客全部源码内容都在 `main` 分支上，而博客的静态网页内容则在 `gh-pages` 分支上。这样做的好处是，我可以在 `main` 分支上进行博客的编辑和维护，而 `gh-pages` 分支则可以作为一个单独的静态网页仓库，用于部署到GitHub Pages上。

### 使用GitHub Actions全自动化部署过程

Hexo 本身其实也有提供 deploy 命令用于部署仓库，但是这个命令每次都要手动执行，稍微有点不太方便。故我选择了尝试使用 GitHub Actions 来实现自动化部署。

GitHub Actions 是处于 git 仓库中 `.github/workflows` 文件夹下的一个 yml 配置文件。其记载了在什么样的条件下，自动执行什么样的操作。

我们的文件是这样的：

```yml 
name: publish-webpage

on:
  push:
    branches:
      - main

jobs:
  publish-webpage:
    runs-on: ubuntu-latest
    if: github.event.repository.owner.id == github.event.sender.id
    steps:
      - name: Checkout source
        uses: actions/checkout@v3
        with:
          ref: main
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 16
      - name: Setup Hexo
        env:
          ACTION_DEPLOY_KEY: ${{ secrets.HEXO_DEPLOY_KEY }}
        run: |
          mkdir -p ~/.ssh/
          echo "$ACTION_DEPLOY_KEY" > ~/.ssh/id_rsa
          chmod 700 ~/.ssh
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.email "40999116+jasonren0403@users.noreply.github.com"
          git config --global user.name "Jason Ren"
          npm install hexo-cli -g
          npm install
      - name: Build and Deploy
        run: |
          hexo clean
          hexo generate
          hexo deploy
```

注意，这些代码与[hexo文档](https://hexo.io/zh-cn/docs/github-pages)上的部署代码并不一样，主要的不同点在于官方教程使用了 `actions-gh-pages`，而自己这边使用的是git原生命令来自己建一个部署密钥。

{% note %}
其实个人认为官方这个代码是比较好的，因为其所使用的 `actions-gh-pages` 无需担心部署密钥，`github_token` 是可以自动在工作流中认证的，不是个人访问令牌，除了首次部署外，无需特别配置。

而且，它还注意了npm依赖的缓存，以后要是再次部署的话，无需再费很长时间安装依赖，博客上线速度可以大大加快。
{% endnote %}

生成deploy key的过程如下：

#### 生成Deploy key
1. 在本地运行 `ssh-keygen -t rsa -b 4096 -C "Hexo Deploy Key" -f github-deploy-key -N ""` 生成一个新的密钥对。

	{% asset_img gen-key-1.png %}

2. 在仓库的设置选项卡中，选择“Secrets”（现为“Secrets and variables”），新建名称为“HEXO_DEPLOY_KEY”（这个名称是与刚才工作流配置文件中的`secrets.*`后面的内容是对应的）的密钥，将刚刚生成的公钥对应的私钥的内容复制进去。

	{% asset_img gen-key-0.png %}

	配置完成后，Action Secrets列表（现Actions Secrets and variables中的Repository secrets）中应该会出现一个新的密钥。
	{% asset_img gen-key-2.png %}

3. 转到“Deploy Keys”中，右上角新添加一个部署密钥。

	{% asset_img gen-key-3.png %}

4. 将刚才生成公钥的内容复制进去（以 ssh-rsa 开头），并勾选“Allow write access”。

	{% asset_img gen-key-4.png %}

配置完成后，每次将博客源码推送到 `main` 分支时，GitHub Actions 就会自动执行部署操作。

[^1]: https://hexo.io/zh-cn/docs/github-pages#项目页面
[^2]: 查阅资料得知waline使用了vercel部署了一个webapp，隔离了前端和存储，避免了安全问题


## 访问优化

博客的内容会越来越多，大量的文章和图片难免会拖慢访问速度，hexo及相关的插件也提供了一些优化的方法。

### hexo-all-minifier

这个插件主要用于对博客中生成的 HTML、CSS、JS 和图片文件进行压缩，减少文件及仓库的体积，提高访问速度。

通过npm安装就可以使用：

```bash 
npm install hexo-all-minifier --save
```

然后在 `_config.yml` 中设置

```yaml
all_minifier: true
```

就可以开启插件。从实际使用体验来看，它自带的 js_concator、js_minifier、html_minifier、css_minifier 都可以保持开启，而在开发（撰写文章）时需要暂时关掉 `image_minifier`，因为压缩图片比较费时。

### cache

NexT 从6.0.0版本开始引入了内容生成缓存，顾名思义，在重复生成大量内容时这个选项可以节省下大量的时间。在 `_config.next.yml` 中完成这个配置即可：

```yaml 
cache:
  enable: true
```
