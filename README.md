# blog-of-jason ![使用 Hexo 构建](https://img.shields.io/badge/hexo-%230E83CD?label=使用%20Hexo%20构建)

这里是 jason 的博客源码仓库，部署的网页源码在隔壁分支 `gh-pages` 上，使用 [hexojs](https://hexo.io/zh-cn/) 的 [hexo-theme-next](https://theme-next.js.org/) 主题构建而成。

[hexo-theme-next 仓库](https://github.com/next-theme/hexo-theme-next)

## 备忘命令

* `hexo new [layout] <title>` 新建文章，可指定 `-p` 文章路径
	- `hexo new draft <文件名>` 新建草稿
* `hexo publish [layout] <filename>` 发表草稿

## Build 失败备忘

`npm run build`（`hexo generate`）曾报：

```text
FATAL Something's wrong...
ReferenceError: window is not defined
  at BrowserPangu.isGridOrFlexContainer
  at ... hexo-pangu/lib/filter.js
```

**原因：** `hexo-pangu@0.6.0` 在 Node 侧用 `vm` 加载 `pangu@9` 的浏览器 UMD（`pangu.umd.js`），上下文未注入全局 `window`；`pangu` 内部调用 `window.getComputedStyle` 时崩溃。

**处理：** 用 `patch-package` 修补 `node_modules/hexo-pangu/lib/filter.js`，在 VM sandbox 中提供 `window` / `globalThis` / `self`；补丁文件为 `patches/hexo-pangu+0.6.0.patch`，`postinstall` 会自动应用。
