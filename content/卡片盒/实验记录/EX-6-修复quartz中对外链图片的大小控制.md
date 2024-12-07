---
title: 修复quartz中对外链图片的大小控制
date: 2024-12-07 14:16
tags:
  - 实验记录
status: 活动
public: true
---
淦！怎么这个锤头 `quartz` **仅仅支持 `obsidian` 中的双链图片调整宽高**啊！！蚌埠住了……

参考了一下发现它对于双链链接的图片的处理是在 `ofs.ts`（[ObsidianFlavoredMarkdown](https://quartz.jzhao.xyz/plugins/ObsidianFlavoredMarkdown)） 中。因为咱们默认应用的就是 obsidian 风格的插件嘛。

这是它匹配双链语法的部分：
```ts
// !? -> optional embedding
// \[\[ -> open brace
// ([^\[\]\|\#]+) -> one or more non-special characters ([,],|, or #) (name)
// (#[^\[\]\|\#]+)? -> # then one or more non-special characters (heading link)
// (\\?\|[^\[\]\#]+)? -> optional escape \ then | then one or more non-special characters (alias)
export const wikilinkRegex = new RegExp(
/!?\[\[([^\[\]\|\#\\]+)?(#+[^\[\]\|\#\\]+)?(\\?\|[^\[\]\#]+)?\]\]/g,
)
```

有样学样写一个匹配正常 markdown 外链的，因为我仅仅是想要将 alias 解析为长宽，所以就匹配 `![name|width](url)` 和 `![alt|widthxheight](url)` 啦～：

这里得参考一下官方的这个图：
![Pasted image 20241205194608.png](https://cdn.sockingpanda.com/f1c2431943c6377bf1e5816fd196539e.png)

也就是说， `markdown` 文件优先在 TransFormers 处理。这部分大致有四种类型的：
- `textTransform`：
	- text -> text
	- 将文件解析为[Markdown AST](https://github.com/syntax-tree/mdast) _之前_ 执行文本到文本的转换
- `markdownPlugins`：
	- markdown -> markdonw
	- 定义了一个[remark插件](https://github.com/remarkjs/remark/blob/main/doc/plugins.md)列表（`remark`是一个以结构化方式将 Markdown 转换为 Markdown 的工具）
- `htmlPlugins`：
	- html -> html
	- 定义了[rehype 插件](https://github.com/rehypejs/rehype/blob/main/doc/plugins.md)列表（与`remark`工作原理类似， `rehype`是一个以结构化方式将 HTML 转换为 HTML 的工具）
- `externalResources`：可能需要在客户端加载才能正常工作的任何外部资源

```ts
export type QuartzTransformerPluginInstance = {
  name: string
  textTransform?: (ctx: BuildCtx, src: string | Buffer) => string | Buffer
  markdownPlugins?: (ctx: BuildCtx) => PluggableList
  htmlPlugins?: (ctx: BuildCtx) => PluggableList
  externalResources?: (ctx: BuildCtx) => Partial<StaticResources>
}
```


这里有个问题啊，貌似 markdown 原生语法是没办法直接在文本中识别、应用图片的宽高的，所以没办法在 `textTransform` 类型的部分处理。

>[!tip]- 碎碎念
>虽然有点了解插件，但这个执行顺序给我看懵逼了哈哈哈哈～
>
>太久没有在前端应用上debug了，这一点小玩意儿花了我好几个小时！！
>
>太菜了！没能和GPT老哥聊明白～

考虑到咱们的 obsidian 本身就支持类似于 `[alt|widthxheight](url)` 这种做法去限制图片大小，就直接在 `ofs.ts` 中加了对这部分的处理。

```ts title="quartz/plugins/transformers/ofs.ts" showLineNumbers{382}
}


visit(tree, "image", (node) => {// [!code ++]
// 检查 alt 是否为有效字符串// [!code ++]
if (typeof node.alt === "string" && node.alt.includes("|")) {// [!code ++]
  // 匹配 alt 中的尺寸信息 (width 或 widthxheight)// [!code ++]
  const match = node.alt.match(/\|(?<width>\d+)(?:x(?<height>\d+))?$/);// [!code ++]
  if (match?.groups) {// [!code ++]
	const { width, height } = match.groups;// [!code ++]
// [!code ++]
	// 修改 hProperties，添加 width 和 height// [!code ++]
	node.data = node.data || {};// [!code ++]
	node.data.hProperties = {// [!code ++]
	  ...node.data.hProperties,// [!code ++]
	  width: width || "auto", // 如果没有宽度，设置为 "auto"// [!code ++]
	  height: height || "auto", // 如果没有高度，设置为 "auto"// [!code ++]
	};// [!code ++]
// [!code ++]
	// 去除 alt 中的尺寸信息// [!code ++]
	node.alt = node.alt.replace(/\|(?<width>\d+)(?:x(?<height>\d+))?$/, "").trim();// [!code ++]
  }// [!code ++]
}// [!code ++]
});// [!code ++]
  
mdastFindReplace(tree, replacements)
```

^cc0e57

>[!success]+ 看各种形状的快乐小狗！
>![P-14.jpg|100](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg) ![P-14.jpg|80](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg)
>
>![P-14.jpg|60](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg) ![P-14.jpg|120](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg)
>
>![P-14.jpg|100x120](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg) ![P-14.jpg|20x100](https://cdn.sockingpanda.com/61433fa13b27a2e71322687ef1c99186.jpg)
>```
>`![dog.jpg|100](https://***.dog.jpg)` `![dog.jpg|80](https://***.dog.jpg)`
>
>`![dog.jpg|60](https://***.dog.jpg)` `![dog.jpg|120](https://***.dog.jpg)`
>
>`![dog.jpg|100x120](https://***.dog.jpg)` `![dog.jpg|20x100](https://***.dog.jpg)`
>```
