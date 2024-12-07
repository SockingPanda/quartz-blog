---
title: QuartzBlog
date: 2024-11-29 10:08
tags:
  - 探索历程
aliases:
  - EP-2
status: 活动
public: true
---
## 简介

记录一下自己基于 `quartz` 搭建博客的过程 

## 正文

---
（2024-11-29) -->

新建项目！准备记录一下我的 QuartzBlog 的搭建，以及整个过程中遇到的问题呀之类的。

首先，这个项目是基于 [quartz](https://github.com/jackyzha0/quartz) 项目为基础构建，后续一定会根据自己的需求去修改、添加插件以实现我的需求。

首先来看[官方的文档](https://quartz.jzhao.xyz/)
- 克隆、下载项目
- 下载包
- 编译、运行

```bash
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm i
npx quartz create
```

---
（2024-12-04) -->

新签出一个分支做自己的修改，避免更新以后下拉冲突。

上面的 `npx quartz create` 里面有3中放文章的方法：
- 空
- 复制
- 软链接

上传我的内容之前，我肯定是需要在 quartz 中查看当前的展示效果的，而且我也不希望总是去手动复制来增加我的管理开销，于是我是用软链接去链接 content 和我要上传的知识库文件夹。

因为要上传的部分在知识库中是不同的文件夹嘛，受到它软链接的启发，我 `unlink` 了 content ，把我需要上传的文件夹分别 `ln -s` 链接进来，以及主页的 `index.md` 单独链接过去。这样我的知识库中就不需要做任何的调整、修改。

`quartz` 在 build 的时候会自动将 `content` 中，软链接链接的文件丢进去 build 出一个个 html，相当不错！

考虑到我的文章中是有很多图片的（正常截图 or 表情包），这些东西貌似直接放到 dist 作为项目丢上去有点不太好，于是我决定给它丢到我的 cloudflare 中的免费 R2 中。

既然图片会丢到 cloudflare 中，那势必需要修改文章中图片的链接，将其指向 R2。所以这里在刚才新分支的基础上又签出了一个新的Release分支，用于做 build 和 上传。

现在则需要考虑在什么地方去遍历、上传图片。

我的表情包呀、其他图片一类的东西，在我的文章中是使用双链链接的。表情包是我自己写的插件插入，它是类似于这样的格式：`[[文件夹/1.png]]`。也就是说，如果我先移动文章再去上传，路径发生变化，可能无法正确检索到 `文件夹/` 这么一个路径。所以必须是在转移文件之前去做这部分处理。

为此写了一个脚本负责遍历我待上传的所有文件（文本中属性内有是否公开属性）。
- 将 quartz 分支切换为 Release 分支
- 以当前文件夹为所有图片做索引（最短形式、相对路径形式、根路径形式）
- 遍历文件夹找属性中 `public` 为 true 的文章
	- 复制文章至目标文件夹
	- 处理文章中所有图片（）
		- 上传
		- 拿到 url
		- 替换


我本地的知识库也是用 `git` 管理，每当我本地知识库 `post` ，就调用上述脚本完成 `Release` 分支切换、全部待上传部分的文件复制，并且 `build quartz` 。

至此，`quartz` 的发布部分的逻辑就讲完了。

---
（2024-12-05) -->


现在的话需要首先去 R2 Buttle 去配置内容。

部署需要的资源上传呀、复制呀之类的完成了！

蚌埠住了，这个锤头不支持对外链图片的大小控制……
![[EX-6-修复quartz中对外链图片的大小控制|修复quartz中对外链图片的大小控制]]


上面代码块部分的效果部分（额外的 `shikijs transform` 部分），灵感来源来源于 [icepro\`s blog：为 quartz 添加额外 shikijs transform](https://iceprosurface.com/%E6%9D%82%E8%AE%B0/%E5%8D%9A%E5%AE%A2%E5%BC%80%E5%8F%91%E4%B8%8E%E7%BB%B4%E6%8A%A4/%E4%B8%BA-quartz-%E6%B7%BB%E5%8A%A0%E9%A2%9D%E5%A4%96-shikijs-transform)。具体使用方式可以参考这个： `@shikijs/transformers` [文档地址](https://shiki.style/packages/transformers) 。

---
（2024-12-06) -->

重新整理了一下我的图片检索、上传、替换脚本。

![[EX-5-obsidian知识库公开部分迁移、图片上传、链接修正|obsidian知识库公开部分迁移、图片上传、链接修正]]


---
（2024-12-07) -->

尝试部署了！！！

咱只要把生成的dist文件丢上去就好了～先去看看官方教程！

恩……果然还是直接丢上去让它来构建最轻松了……


正在配置 `cloudflare` ，
![Pasted image 20241207162221.png](https://cdn.sockingpanda.com/public/3e65247a77b0fac867ea97fd0e7812b2.png)


