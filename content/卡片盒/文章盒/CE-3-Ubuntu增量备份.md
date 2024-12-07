---
title: Ubuntu增量备份
create: 2024-12-01 14:41
tags:
  - 文章
aliases: 
public: true
---

## 前言

昨天晚上看着手机突发奇想，准备尝试搞搞让手机作为扩展屏扩展一波。看到这篇文章想必就知道我改图形界面配置搞崩了！![P-28.jpg|100](https://cdn.sockingpanda.com/a9b8e62e3a31d335ddfde1a2e6f47e55.jpg)

第一次就是加了这两行：

![8113e45b6f34555cf6b0dbc86625dfb5.jpg](https://cdn.sockingpanda.com/80525c9aacb3c0c80d4df7f59e082c0d.jpg)
怕是给我主屏幕搞到外面那个虚拟屏幕了……因为这个锤子文件我记着是通过 `nvidia-xconfig` 创建出来的，遂给它直接删了，然后重启就好了～

这时候 bro 还觉得自己贼牛逼，在朋友表示 “老实点得了” 之后，又去尝试了一次其他方式（指同一个文件换了一部分配置去修改）。

然后，然后就没有然后了……删除 `xorg.conf` 都没得救了……并且最痛苦的是，越是自动修复，越是炸！！！![P-21.jpg|100](https://cdn.sockingpanda.com/48429c6f1f6b0dc1886332cc0ba3f78c.jpg)

![4d3453bb11592ee9189120a615f57294.jpg](https://cdn.sockingpanda.com/6965778e22072ec3ce613e9d603a068d.jpg)

心路历程展示：

![Pasted image 20241201145519.png](https://cdn.sockingpanda.com/e8a730f99a9fa1d30a6941b679e525da.png)

好在十几天前用 `diskGenious` 分盘的时候有“顺便”为 Ubuntu 系统盘做备份，重要的数据也不在那个盘里～！！![P-29.png|100](https://cdn.sockingpanda.com/aa03b83afeb187cee2f45405185bd83e.png)


坏消息是，我脑子抽了，没有把一些数据一类的文件从坏掉的 Ubuntu 盘里面拿出来，直接执行了恢复……![P-3.jpg|100](https://cdn.sockingpanda.com/1d61010deaaf699c7323892b55621729.jpg)

为了避免二次出现这种锤头事情，本喵决定探索一下系统的增量备份方案～
![P-12.jpg|100](https://cdn.sockingpanda.com/7fd62de3ffa5b7a4318a08cc6109db92.jpg)


## 需求

我是 `ext4` 的盘做 Ubuntu 的系统盘，数据考虑到使用场景放在 `NTFS` 类型的盘中。所以我需要一个备份方案能够将我的 `ext4` 盘中的数据保存进 `NTFS` 盘中。

## Deja Dup


![[EX-2-基于“Deja Dup”的增量备份123|基于“Deja Dup”的增量备份]]

## Timeshift

![[EX-3-基于“Timeshift”的增量备份|基于“Timeshift”的增量备份]]
## Duplicity

![[EX-4-基于“Duplicity”的增量备份|基于“Duplicity”的增量备份]]

## rsync

>[!failure] rsync 备份内容有问题，至少我备份的那几个文件夹直接恢复是无效的 【[[CE-4-引导项失效恢复|详见下一篇文章]]】

>[!success] 这玩意儿就是前面`timeshift` 用的工具……被前面误导，导致直到最后才尝试去看它，发现可以存到 `NTFS文件系统` 中

![[EX-1-基于“rsync”的增量备份]]

## 结论

>[!example] 结论
>- 如果只是个人日常备份到 linux文件系统的话，可以直接使用 `Timeshift` 省时省力～
>- 如果是愿意折腾，期望更精细控制备份的话，可以直接尝试 `rsync` 。
>- 不建议去尝试 `Duplicity` ！

