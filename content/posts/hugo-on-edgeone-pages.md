---
title: "使用 EdgeOne Pages 托管Hugo博客"
categories: [ "cdn" ]
tags: [ "edgeone" ]
draft: false
slug: "hugo-on-edgeone-pages"
date: "2025-07-11T12:30:00+08:00"
---

自从腾讯云客服给我打电话说我的备案域名解析到了境外IP上让我整改，已经过去了两年。

当时我把解析到 Github Pages 的域名更改到了轻量云，并且把 Github Pages 打包下来使用 Caddy 搭建了静态网站，由于更新不便，我再也没有更新过博客。

前段时间白嫖了 EdgeOne 免费 CDN 测试加速 Github Pages，说实话并不好用。

首先第一次请求回源到 Github 特别慢，大概率是用了国内的节点去请求；其次 CDN 的缓存加速功能，我更新了博客的内容就需要手动把 CDN 的缓存全部清空才行，不缓存的话又起不到加速的作用。

今天忙里偷闲，测试一下EdgeOne Pages 托管Hugo博客，EdgeOne Pages 是支持 Node.js 相关的框架预设，我以为托管 Hugo 博客会很麻烦，结果发现真的很简单。

首先，你要先把 Hugo 托管到 Github Pages 上，这步教程很多，我就不再赘述。

然后，你要在 EdgeOne 上创建一个 Pages 项目，选择你的 Github Pages 仓库，记得分支选择编译后文件所在的 `gh-pages` 然后选择 Other 预设，其他不需要改变。

![](/images/edgeone-pages-deploy.png)

最后点击开始部署，等待部署完成即可。

----

每次提交后，Github Action 自动编译发布在分支 `gh-pages` 上， EdgeOne 会自动检测到仓库 `gh-pages` 分支的变化，最后更新到 EdgeOne Pages 上。
