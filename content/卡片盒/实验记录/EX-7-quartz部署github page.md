---
title: quartz部署github page
date: 2024-12-07 23:59
tags:
  - 实验记录
status: 已完成
public: true
---


## 部署 `github page` 及 自动化构建

 `github repo` 中的 `setting` 里面的 `page` 项，添加 `action` 。
![Pasted image 20241207234913.png](https://cdn.sockingpanda.com/efbec3d2df537ec3989b2b7b63553a73.png)

`action` 部分是参考[官方的教程](https://quartz.jzhao.xyz/hosting#github-pages)，大致就是：读哪个分支，在什么环境、工具构建，最终产物放在哪儿。

```yml title=".github/workflows/depoly.yml"
name: Deploy Quartz site to GitHub Pages
 
on:
  push:
    branches:
      - Release # 触发条件修改为 Release 分支
 
permissions:
  contents: read
  pages: write
  id-token: write
 
concurrency:
  group: "pages"
  cancel-in-progress: false
 
jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Fetch all history for git info
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install Dependencies
        run: npm ci
      - name: Build Quartz
        run: npx quartz build
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public
 
  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

玩事儿了以后，这个 `on.push.branches` 的分支一旦更新，就会自动完成后续工作，完成自动化构建。

## 添加自定义域名

这里得保证你自己有自己的域名哇～没有的话可以忽略这一步了～

[教程链接](https://quartz.jzhao.xyz/hosting#custom-domain)

在自己的 `DNS` 上加上：
- 185.199.111.153
- 185.199.110.153
- 185.199.109.153
- 185.199.108.153

还得把代理状态改成 `仅DNS`（橙色的改成灰色的）
![Pasted image 20241207162221.png](https://cdn.sockingpanda.com/3e65247a77b0fac867ea97fd0e7812b2.png)
