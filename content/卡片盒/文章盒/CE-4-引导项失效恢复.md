---
title: 引导项失效恢复
date: 2024-12-03 08:49
tags:
  - 文章
aliases: 
public: true
---
## 前言

好家伙好家伙，前天才写的[[CE-3-Ubuntu增量备份|备份文章]]，昨天就出问题了！![P-28.jpg|100](https://cdn.sockingpanda.com/a9b8e62e3a31d335ddfde1a2e6f47e55.jpg)

因为代码中有 `mkdir -p` 没有文件夹会自动创建文件夹嘛，所以它在前面的时候会占用我硬盘本身要挂载的位置，导致硬盘的位置发生变化。本着 *“service部分已经做到了开机以后、硬盘插入以后才被运行”* 这么一个想法，果断给前面的 `mkdir` 删了……

[[EX-1-基于“rsync”的增量备份#^9bca9c]]

```bash
# 定义备份目录
BACKUP_DIR="/to/your/path/backup"
BACKUP_TYPE_DIR="$BACKUP_DIR/system_incremental"

# 定义系统文件备份目录
SYSTEM_DIRECTORIES="/etc /bin /lib /usr /sbin"

# 确保备份目录存在
mkdir -p $BACKUP_TYPE_DIR // [!code --]

```

重启以后经历了长时间黑屏，实在是稳不住了就尝试强行关机。再次开机，无论是 `windows` 还是 `ubuntu` 引导项都消失了！![P-13.jpg|100](https://cdn.sockingpanda.com/94e57211d36a86821061a7400d7c6ae7.jpg)

![f6b5e73d8ac6b83d28ec21ba5cc38190.jpg](https://cdn.sockingpanda.com/d03051f3b29444fab5364f7f5dc51da5.jpg)

事后推测，重启以后脚本自动运行，然后数据写入了未被定义的区域（linux可真是自由啊……）。

>[!question] 如果我没有果断重启，是否会损坏我的其他数据？

>[!danger] 请勿尝试在未知区域写入数据，不是所有库都有防护的措施

由于**不了解引导项** ，外加之前的思维惯性 *系统引导项出问题$\approx$系统出问题* ，还有[[CE-3-Ubuntu增量备份|上篇文章]]中备份了数据的自信，我重装了系统，在新系统中去重新恢复数据。

事实证明，增量备份问题挺大的，我保存的那几个文件夹恢复回去直接给我新装的系统搞卡死了……![P-3.jpg|100](https://cdn.sockingpanda.com/1d61010deaaf699c7323892b55621729.jpg)

综上所述，~~咱又能水一篇文章了！～~~咳咳，开始本文的主题，解决引导项失效问题。

>[!tip] 这里的记录不是我实际尝试的步骤，我自己遇到这事儿的时候比较慌乱……

---
## GRUB 命令行界面手动处理

引导项出问题的话，我这里是会转到 GRUB 命令行界面，如下图：
![Pasted image 20241203201855.png](https://cdn.sockingpanda.com/9a055f5addbce890864e8fab15b5e443.png)

>[!note] 这时候首先应该去确认自己系统的 boot文件是否还在（即下面的步骤）

---
### 引导恢复步骤

1. **查看已加载的磁盘和分区**  
    使用 `ls` 命令，列出当前系统识别到的所有磁盘和分区，便于确认系统分区的位置。
    
2. **确定系统分区**  
    多次使用 `ls` 命令并结合不同路径进行检查，以找到存储操作系统的分区。例如，通过检查分区是否包含 `/boot` 或 `/root` 文件夹来确定系统分区。
    
   >[!tip] 也能通过分区大小来辅助判断
    
3. **设置根路径**  
    根据找到的系统分区位置，使用以下命令设置根路径：
    
    ```bash
    set root=(hd0,gpt1)
    ```
    
> [!example]- 参数说明
> **`hd0`**
> - 表示第一个磁盘。
> 
> **`gpt1`**
> - 表示该磁盘上的 GPT 分区类型的第一个分区。

    （假设系统分区在 `hd0,gpt1`，请根据实际情况调整。）
    
4. **加载 GRUB 配置文件**  
    使用以下命令加载默认的 GRUB 配置文件：
    
    ```bash
    configfile /boot/grub/grub.cfg
    ```
    
    - 该命令会尝试在根路径下的 `/boot/grub/` 目录中找到并加载 `grub.cfg` 配置文件。
    - 例如，在 Ubuntu 系统中，GRUB 配置文件通常位于这个路径。
5. **启动系统**  
    运行以下命令，从加载的 GRUB 配置中启动系统：
    
    ```bash
    boot
    ```
    
    - 这会使用配置文件中的设置启动到之前未损坏的引导项。

---

通过以上步骤，可以手动引导进入系统并正常启动。

## 进系统后的修复


1. **确认当前分区布局** 确认哪个分区当前挂载在 `/boot/efi`：
    
    ```bash
    df -h | grep /boot/efi
    ```
    
2. **备份现有 EFI 分区** 在更改之前，建议先备份当前 EFI 分区的内容（以防止误操作导致系统无法启动）：
    
    ```bash
    sudo cp -r /boot/efi /path/to/backup
    ```
    

因为我们使用的是 UEFI 去启动，所以需要找到一个 `EFI 分区` 作为启动分区。
这里可以使用 `gparted` 来处理分区部分。

如下图所示，右上角部分选择磁盘，然后就能看到分区的标识（比如 boot，esp 意味着这个分区是 EFI 分区）。
![Pasted image 20241203225413.png](https://cdn.sockingpanda.com/e0195979fd99f974c0840d8be675ab21.png)

### 切换分区
我现在的话希望用 sd5 那个磁盘作为启动分区
#### 步骤 1：**任意方式挂载 `/dev/sda5` 到一个临时路径**

将 `/dev/sda3` 挂载到一个临时路径，比如 `/mnt`：

```bash
sudo mount /dev/sda5 /mnt
```

挂载后，检查 `/mnt` 是否为空（如果分区中已经有内容，如 `EFI` 文件夹，可以先清理分区）：

```bash
ls /mnt
```

---

#### **步骤 2：重新安装 GRUB 到 `/dev/sda5`**

告诉 GRUB 使用新的 EFI 分区 `/dev/sda5`：

```bash
sudo grub-install --target=x86_64-efi --efi-directory=/mnt --bootloader-id=Ubuntu --recheck
```

> [!example]- 参数介绍
> **`grub-install`**
> - GRUB 的安装命令，用于安装 GRUB 到指定设备或分区。
> 
> **`--target=x86_64-efi`**
> - 指定目标架构和引导模式。
> - **`x86_64-efi`** 表示安装用于 UEFI（统一可扩展固件接口）的 64 位 GRUB。
> 
> **`--efi-directory=/mnt`**
> - 指定 EFI 系统分区（ESP）的挂载目录。
> - `/mnt` 是一个挂载点，此处假设 EFI 分区已经挂载到 `/mnt`。
> - 在 UEFI 模式下，GRUB 的启动文件需要存放在 EFI 分区中。
> 
> **`--bootloader-id=Ubuntu`**
> - 指定 GRUB 在 EFI 引导管理器中的名称。
> - **`Ubuntu`** 是这个 GRUB 引导项的标识符（出现在 BIOS/UEFI 的启动菜单中）。
> - 这个名字可以是自定义的，比如 "MyLinuxBoot"，但最好与系统名称对应。
> 
> **`--recheck`**
> 
> - 强制重新检测目标设备，确保所有设备和文件系统都正确识别。
> - 通常用于修复或重新安装 GRUB 时。


这会将 GRUB 的引导文件重新写入 `/mnt`，也就是被挂载在 `/mnt` 上的 `/dev/sda5` 分区中。

---

#### **步骤 3：更新 GRUB 配置**

更新 GRUB 配置文件，确保它可以正确检测和配置启动项：

```bash
sudo update-grub
```

---

#### **步骤 4：修改 UEFI 引导顺序**

>[!danger] 下方改动不一定生效。我的 redmiG 貌似是会按照顺序去读取所有的 `EFI` 分区，只能够改顺序，无法通过命令删除引导项


现在，你需要将 `/dev/sda5` 注册为新的 EFI 引导分区：

1. 查看当前引导顺序：
    
    ```bash
    sudo efibootmgr -v
    ```
    
2. 手动添加 `/dev/sda3` 中的 GRUB 文件作为引导项：
    
    ```bash
    sudo efibootmgr --create --disk /dev/sda --part 5 --label "Ubuntu" --loader "\\EFI\\Ubuntu\\grubx64.efi"
    ```

> [!example]- 参数介绍
> **`efibootmgr`**
> 
> - 一个管理 EFI 引导项的工具，用于查看、修改或创建 EFI 系统的引导项。
> 
> **`--create`**
> 
> - 创建一个新的 EFI 引导项。
> 
> **`--disk /dev/sda`**
> 
> - 指定引导项所对应的磁盘。
> - 这里的 `/dev/sda` 是存储 EFI 系统分区的磁盘。
> 
> **`--part 5`**
> 
> - 指定 EFI 系统分区在磁盘上的分区号。
> - 这里是 `/dev/sda5`，表示 EFI 分区位于 `/dev/sda` 的第 5 个分区。
> 
> **`--label "Ubuntu"`**
> 
> - 指定这个引导项在 EFI 引导菜单中的名称。
> - 在 BIOS/UEFI 引导菜单中，这个名称将显示为 "Ubuntu"。
> 
> **`--loader "\\EFI\\Ubuntu\\grubx64.efi"`**
> 
> - 指定引导加载程序的路径（相对于 EFI 分区根目录）。
> - 这里的 `\\EFI\\Ubuntu\\grubx64.efi` 是 GRUB 引导文件的位置。
>     - 双反斜杠是为了兼容 EFI 的文件路径表示方式。
>     - `grubx64.efi` 是 GRUB 的 UEFI 引导文件。


3. 设置新的引导项为优先级最高（也可以直接在 主板的BIOS里面去调整）：
    
    ```bash
    sudo efibootmgr --bootorder XXXX
    ```
    
    （将 `XXXX` 替换为新创建的引导项的编号）。

4. 你也可以删除不需要的引导项：

   >[!danger] 谨慎操作，注意备份

	```bash
	sudo efibootmgr --delete-bootnum --bootnum 0000
	```

> [!example]- 参数介绍  
> 
> **`efibootmgr`**
> 
> - 一个管理 EFI 引导项的工具，用于查看、修改或删除 EFI 系统的引导项。
> 
> **`--delete-bootnum`**
> 
> - 删除指定的引导项。
> - 使用此参数时，必须搭配 `--bootnum` 参数来指定要删除的引导项编号。
> 
> **`--bootnum 0000`**
> 
> - 指定需要删除的引导项编号。
> - `0000` 是引导项的编号，可以通过运行 `sudo efibootmgr` 查看所有引导项并确认编号。
> - 请谨慎操作，确保删除的引导项是无用的或不再需要的。

---

#### **步骤 5：重启并测试**

重启系统，进入 BIOS/UEFI 设置，确保新注册的 `Ubuntu` 引导项位于启动顺序的顶部。保存设置并启动，确认是否能够正常加载 GRUB。

---

### （可选）修改默认 /boot/efi 挂载点

打开 `disk` 选择刚才的 EFI 分区，给它搞成系统启动自动挂载、设定挂载点为 `/boot/efi` 文件系统 `vfat` 。

>[!tip] 需要把其他挂载在 `/boot/efi` 的分区给卸载，且取消它的启动时挂载

>[!bug] 貌似挂载点上面一行不能为空，也不能只是 `defaults`，所以我加了一个名称 `EFI`。


![Pasted image 20241204002453.png](https://cdn.sockingpanda.com/265ff10c5ea791f15f96e87190aabfc8.png)
![Pasted image 20241204001815.png](https://cdn.sockingpanda.com/78240e41e3edda01b990fad771eef0d0.png)


## Windows 引导项恢复

貌似得等我新买的U盘到了以后才能尝试恢复了……手动尝试了用 GRUB命令行引导界面去开，但实在是开不来……明明系统数据都在，仅仅是 EFI 分区的损坏，应该可以和 `Ubuntu` 类似，直接去手动进系统来着。