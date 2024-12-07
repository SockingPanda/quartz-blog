---
title: 基于“Timeshift”的增量备份
create: 2024-12-02 02:10
tags:
  - 实验记录
status: 活动
public: true
---
>[!failure] 无法将快照保存到 `NTFS` 文件系统中

“Timeshift” 的话，使用起来很方便，基本就是傻瓜式操作。打开向导，然后……然后对着看、对这选就对了。

>[!quote] RSYNC 快照
> - 快照是通过使用**rsync**创建系统文件的副本，以及从以前的快照中**硬链接未更改的文件**来创建的。
> - 在创建首个快照时，所有文件都会**被复制**。**后续的快照将在此基础上增加**，未更改的文件会硬链接到以前的快照。
> - 快照可以保存到任一个格式化为Linux文件系统的磁盘。请将快照保存到非系统或外部磁盘，这样即使系统磁盘损坏或重新格式化，也可恢复系统。
> - 您可以排除文件和目录以节省磁盘空间。

它能够快速选择需要保存计划、需要保存的用户、筛选需要保存的文件。

![Pasted image 20241201151844.png](https://cdn.sockingpanda.com/79f2bc839eb587c28a4feac240670449.png)
![Pasted image 20241201151657.png](https://cdn.sockingpanda.com/11cd999573c5e0972bdb361f94a9158a.png)
![Pasted image 20241201151756.png](https://cdn.sockingpanda.com/5408de5a77ffb9ca51ae537e43afdb83.png)

很可惜，它的快照只能够保存在 Linux文件系统的磁盘（sad！！）。
