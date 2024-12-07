---
title: "obsidian知识库公开部分迁移、图片上传、链接修正"
create: "2024-12-07 01:43"
tags:
  - 实验记录
status: 活动
public: true
---
整理一下我自己的 obsidian知识库公开部分迁移、图片上传、链接修正部分内容。

## 知识库部分迁移

使用知识库做记录的时间不多，我自己的知识库是相当乱的！这样的话就容易出现这样的情况：

>[!question] 最终需要的文件夹结构可能来自于多个不同文件夹中的内容

`quartz` 在创建的时候会有3个选项，其中一个就是符号链接你需要的文件夹。这个确实是给了我灵感，于是我也用相同的方式，使用ubuntu 的符号链接链接我需要的文件夹，将其放入 `content` 文件夹内。

这时候又有另一个问题：

>[!question] 知识库中通过双链链接资源文件，而站点中的资源文件应该向对象存储请求

也就是说，我**不能直接使用符号链接**来链接我的文件，而是需要将其原文件修改、文中的双链都需要**改用常规 `markdown 语法`**，并且资源链接指向 `对象存储OSS` 中。

为此，我在我自己的知识库中的顶层文件夹中新增了一个 `release` 文件夹，并且**将需要上传的文件夹的符号链接放进去**～这样知识库迁移只需要用脚本遍历知识库的 `release` 文件夹就能够做到了！

要支持符号链接的话，需要在 `os.walk()` 中添加参数 `followlinks=True`
```python {4}
# 查找属性 public 为 true 的文章
def find_public_articles(source_folder):
    public_articles = []
    for root, dirs, files in os.walk(source_folder, followlinks=True):
        for file in files:
            if file.lower().endswith('.md'):
                file_path = os.path.join(root, file)
                with open(file_path, 'r', encoding='utf-8') as f:
                    try:
                        content = f.read()
                        front_matter_match = re.match(r'^---\n(.*?)\n---\n', content, re.DOTALL)
                        if front_matter_match:
                            front_matter = yaml.safe_load(front_matter_match.group(1))
                            if front_matter and front_matter.get('public', False):
                                public_articles.append(file_path)
                    except yaml.YAMLError as e:
                        print(f"YAML 解析错误在 {file_path}: {e}")
                        continue
    return public_articles
```

## 图片上传

考虑到我的图片引用有些是相对根路径，有些是最短路径，这里获取一下文件的最短路径和相对根路径

```python
def map_files_to_shortest_and_relative_paths(source_folder):
    file_mapping = {}

    for root, dirs, files in os.walk(source_folder, followlinks=True):
        for file in files:
            # 获取文件的完整路径
            file_path = os.path.join(root, file)
            
            # 获取文件相对源文件夹的路径
            relative_path = os.path.relpath(file_path, source_folder)
            
            # 获取文件的最短路径（即文件名）
            shortest_path = file  # 文件名作为最短路径
            
            # 将最短路径和相对路径映射到实际路径
            file_mapping[shortest_path] = file_path
            file_mapping[relative_path] = file_path
    
    return file_mapping
```

上面已经拿到了所有需要的公开的文件，我们需要的是在文件中找到所有双链部分，将指向资源文件的部分上传。

>[!tip] 嘛，主要是不希望公开的资源和自己私有的资源混合在一起

```python {10}
   def process_markdown_files_for_upload(self, resource_map, markdown_files, source_folder, map_dict):
        """
        处理一组 Markdown 文件，上传图片并构建本地路径到远端 URL 的映射。
        """
        for file_path in markdown_files:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()

            # 使用正则表达式查找所有 ![[资源名称]] 或 ![[资源名称|xxx]]
            pattern = re.compile(r"(?P<image_marker>!?)(?:\[\[(?P<resource>[^\|\]]+)(?:\|(?P<alias>[^\]]+))?\]\])")
            
            matches = pattern.finditer(content)

            if not matches:
                print(f"未找到图片: {file_path}")
                continue
            
            for match in matches:
                resource_name = match.group("resource")
                # 查找 image_map 中的实际路径
                if resource_name in resource_map:
                    actual_path = resource_map[resource_name]
                    print(f"资源 {resource_name} 对应的实际路径: {actual_path}")
                else:
                    print(f"未找到资源 {resource_name} 对应的路径映射")
                    continue

                if not os.path.isfile(actual_path):
                    print(f"图片文件不存在: {actual_path}")
                    continue

                # 上传图片到 R2
                uploaded_url = self.r2_client.upload_image(actual_path)
                if uploaded_url:
                    # 使用 Unix 风格路径作为键
                    relative_unix_path = convert_to_unix_path(resource_name)
                    map_dict[relative_unix_path] = uploaded_url
                    print(f"已上传 {actual_path} 至 {uploaded_url}")
                else:
                    print(f"上传失败: {actual_path}")
```

将资源上传以后，复制文件。

>[!tip] 为了防止资源文件仅仅修改了名字重复上传，这边是将上传的文件名替换为了它的 MD5
>`cloud flare` 的 `R2 oss` 是支持给文件加属性进去的，也可以直接加属性达到相同的效果啦～俺就是嫌麻烦了



## 链接修正

这部分就重新匹配啦～匹配完以后，根据存下来的映射关系做替换就好了～

大概就下面这些是重点吧，将资源文件的双链替换为正常markdown支持的语法，并且保留 `[name|alias](url)` 中 `alias` 对图片大小的控制（外链图片控制是在 `quartz` 中修改的，[[EX-6-修复quartz中对外链图片的大小控制|详见此文]]）
```python
def replace_match(match):
	# 提取通用部分
	resource_name = match.group("resource")
	alias = match.group("alias")
	is_image = bool(match.group("image_marker"))
	
	# 获取 URL 映射路径
	url = image_url_mapping.get(resource_name, resource_name)
	resource_name = resource_name.split("/")[-1]
	# 判断是图片还是普通链接
	if is_image:
		if alias:
			# 检查 alias 是否是指定格式（纯数字 或 数字x数字 或 x数字）
			if re.fullmatch(r"(\d+|\d+x\d+|x\d+)", alias):
				return f"![{resource_name}|{alias}]({url})"
			else:
				return f"![{alias}]({url})"
		else:
			return f"![{resource_name}]({url})"
	else:
		if alias:
			return f"[{alias}]({url})"
		else:
			return f"[{resource_name}]({url})"

# 定义正则表达式，捕获所有双链格式
pattern = re.compile(
	r"(?P<image_marker>!?)(?:\[\[(?P<resource>[^\|\]]+)(?:\|(?P<alias>[^\]]+))?\]\])"
)
new_content = pattern.sub(replace_match, content)   
```

>[!tip]- 碎碎念
>淦，写着写着就感觉不太对了，貌似我没必要刷两次文本！
>
>也就是说，直接复制文件，然后用知识库的map刷一遍就好了……
>
>下次再说～

