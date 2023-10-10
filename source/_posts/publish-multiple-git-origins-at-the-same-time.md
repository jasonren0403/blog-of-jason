---
title: 同时推送多个 Git 仓库提交
date: 2023-10-10 11:28:11
tags:
  - Git
  - VCS
categories:
  - ["版本控制", "Git"]
excerpt: Git 小技巧记录
---

在使用 Git 时，有一种场景是需要将代码同时推送到多个仓库中，分别进行提交、推送也勉强可以，但是太不优雅了！经过简单的资料查阅，我发现 Git 本身其实提供了这样的命令来完成这项任务。

在多方对比后，我选择了“一次推送”的方案。直接修改了项目本地配置文件 `.git/config`，在 `remote "origin"` 做了如下设置：

```text
[remote "origin"]
	url = github地址
	url = gitee地址 # 新增
	fetch = +refs/heads/*:refs/remotes/origin/*
```

{% note %}
`git remote set-url --add origin <newUrl>` 的实质应该也是在配置文件中增加配置。
{% endnote %}

然后，在 ssh 配置文件 `.ssh/config` 配置了两个网站的公钥访问：

```text
# gitee
Host gitee.com
    Port 22
    HostName gitee.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_ed25519_gitee
    User git
# github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_ed25519
    User git
```

这样，推送的时候只需 `git push` 便可同时完成两个仓库的推送。

## 参考资料

* [Git Ebook：远程仓库的使用](https://git-scm.com/book/zh/v2/Git-基础-远程仓库的使用)
* [将一个项目同时从本地推送到 GitHub 和 Gitee](https://www.cnblogs.com/poloyy/p/12215199.html)