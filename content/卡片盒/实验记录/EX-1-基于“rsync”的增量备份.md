---
title: 基于“rsync”的增量备份
create: 2024-12-01 14:40
tags:
  - 实验记录
aliases: 
status: 已完成
public: true
---
## 简介

`rsync` 是一个功能强大的文件同步工具，支持高效的数据传输。它能够：

- 进行全量或增量备份。
- 支持增量备份，这意味着只有自上次备份以来发生变化的文件才会被传输。
- 提供 `--link-dest` 选项，支持硬链接（hard link），让你能高效地存储多个备份版本。

**增量备份的关键：**
- `rsync` 通过[[K-11-硬链接|硬链接]]（`--link-dest`）避免重复存储没有变化的文件，显著节省存储空间。

>[!tip] 增量备份是在前一次备份的基础上去做的

## **全量备份**

`rsync` 全量备份：
```bash
sudo rsync -a --progress --delete /etc /bin /lib /usr /sbin /to/your/path/backup/system_backup/
```
- `-a` ：**归档模式**
	- **递归复制目录**：会复制整个目录，包括其中的子目录和文件。
	- **保留文件属性**：保持源文件的权限、时间戳、符号链接、用户和组信息等。
	- **保留设备文件**（对于特殊设备文件和管道等）。
	- **保留符号链接**：如果源是符号链接，`rsync` 会保留链接本身，而不是复制链接指向的内容。
- `--progress` ：显示文件传输的进度信息（每个文件的传输进度、已复制的数据量、传输速度等）。
- `--delete` ：删除目标目录中那些源目录中不存在的文件。常用于保持源和目标目录的完全同步。

## **脚本实现全量备份和增量备份**

我们编写了一个 Bash 脚本来自动化全量和增量备份的过程，这里我仅保留最后3次的记录。

核心步骤如下：

```bash
#!/bin/bash

# 定义备份目录
BACKUP_DIR="/to/your/path/backup"
BACKUP_TYPE_DIR="$BACKUP_DIR/system_incremental"

# 定义系统文件备份目录
SYSTEM_DIRECTORIES="/etc /bin /lib /usr /sbin"

# 确保备份目录存在
mkdir -p $BACKUP_TYPE_DIR

# 获取当前日期（格式：YYYYMMDD）
CURRENT_DATE=$(date +%Y%m%d)

# 获取当天之前的所有备份（按日期分组）
PREVIOUS_BACKUPS=$(ls -d $BACKUP_TYPE_DIR/* | grep -v "$CURRENT_DATE" | sort)

# 对于今天之前的每一天，只保留最新的备份
if [ ! -z "$PREVIOUS_BACKUPS" ]; then
    echo "Cleaning up previous backups..."
    # 获取每一天的最新备份
    DAYS=$(echo "$PREVIOUS_BACKUPS" | awk -F- '{print $1"-"$2"-"$3}' | sort | uniq)
    for day in $DAYS; do
        # 获取该天的所有备份
        DAY_BACKUPS=$(echo "$PREVIOUS_BACKUPS" | grep "$day")
        # 获取当天最新的备份（保留最新的一个）
        KEEP_BACKUP=$(echo "$DAY_BACKUPS" | tail -n 1)
        # 删除当天其他的备份
        echo "$DAY_BACKUPS" | grep -v "$KEEP_BACKUP" | xargs sudo rm -rf
    done
fi

# 获取最新的备份目录
LATEST_BACKUP=$(ls -d $BACKUP_TYPE_DIR/* | sort | tail -n 1)

# 如果没有任何备份，做全量备份
if [ -z "$LATEST_BACKUP" ]; then
    NEW_BACKUP="$BACKUP_TYPE_DIR/full-$CURRENT_DATE-$(date +%H%M%S)"
    sudo rsync -a --progress $SYSTEM_DIRECTORIES $NEW_BACKUP
else
    # 做增量备份
    NEW_BACKUP="$BACKUP_TYPE_DIR/inc-$CURRENT_DATE-$(date +%H%M%S)"
    sudo rsync -a --progress --link-dest=$LATEST_BACKUP $SYSTEM_DIRECTORIES $NEW_BACKUP
fi

# 删除多余的旧备份（保留最近3天的每一天的最后一次备份）
cd $BACKUP_TYPE_DIR
ls -d * | sort | uniq -w 8 | head -n -3 | xargs sudo rm -rf
```

^9bca9c

这里还有一个备份用户数据的， `--exclude` 是排除了 `*.log` 和 `*.cache` 这种文件减少错误：
```diff
#!/bin/bash

# 定义备份目录
BACKUP_DIR="/to/your/pathT7_Touch/backup"
- BACKUP_TYPE_DIR="$BACKUP_DIR/system_incremental"
+ BACKUP_TYPE_DIR="$BACKUP_DIR/user_incremental"

# 定义需要备份的目录（假设是 /home 和 /var）
FILES_TO_BACKUP="/home /var"

# 确保备份目录存在
mkdir -p $BACKUP_TYPE_DIR

# 获取当前日期（格式：YYYYMMDD）
CURRENT_DATE=$(date +%Y%m%d)

# 获取当天之前的所有备份（按日期分组）
PREVIOUS_BACKUPS=$(ls -d $BACKUP_TYPE_DIR/* | grep -v "$CURRENT_DATE" | sort)

# 对于今天之前的每一天，只保留最新的备份
if [ ! -z "$PREVIOUS_BACKUPS" ]; then
    echo "Cleaning up previous backups..."
    # 获取每一天的最新备份
    DAYS=$(echo "$PREVIOUS_BACKUPS" | awk -F- '{print $1"-"$2"-"$3}' | sort | uniq)
    for day in $DAYS; do
        # 获取该天的所有备份
        DAY_BACKUPS=$(echo "$PREVIOUS_BACKUPS" | grep "$day")
        # 获取当天最新的备份（保留最新的一个）
        KEEP_BACKUP=$(echo "$DAY_BACKUPS" | tail -n 1)
        # 删除当天其他的备份
        echo "$DAY_BACKUPS" | grep -v "$KEEP_BACKUP" | xargs sudo rm -rf
    done
fi

# 获取最新的备份目录
LATEST_BACKUP=$(ls -d $BACKUP_TYPE_DIR/* | sort | tail -n 1)

# 如果没有任何备份，执行全量备份
if [ -z "$LATEST_BACKUP" ]; then
    NEW_BACKUP="$BACKUP_TYPE_DIR/full-$CURRENT_DATE-$(date +%H%M%S)"
-    sudo rsync -a --progress $SYSTEM_DIRECTORIES $NEW_BACKUP
+    sudo rsync -a --progress --exclude="*.log" --exclude="*.cache" $FILES_TO_BACKUP $NEW_BACKUP
else
    # 做增量备份
    NEW_BACKUP="$BACKUP_TYPE_DIR/inc-$CURRENT_DATE-$(date +%H%M%S)"
-    sudo rsync -a --progress --link-dest=$LATEST_BACKUP $SYSTEM_DIRECTORIES $NEW_BACKUP
+    sudo rsync -a --progress --link-dest=$LATEST_BACKUP --exclude="*.log" --exclude="*.cache" $FILES_TO_BACKUP $NEW_BACKUP
fi

# 删除多余的旧备份（保留最近3天的每一天的最后一次备份）
cd $BACKUP_TYPE_DIR
ls -d * | sort | uniq -w 8 | head -n -3 | xargs sudo rm -rf

```


>[!bug] rsync error: some files/attrs were not transferred (see previous errors) (code 23) at main.c(1338) [sender=3.2.7]
>备份的内容太多了！咱也不知道是哪个的问题，所幸就不管它了～

## 设置服务随着开机且目标硬盘接入启动

这部分主要是为我的系统备份准备的

### 创建服务
在 `/etc/systemd/system/` 目录下创建一个 `service` 。比如：
```bash
sudo nano /etc/systemd/system/system_backup_after_mount.service
```

>[!tip] 我这里是把脚本放到了 `/usr/local/bin/` 下，所以可以sudo全局调用。

```
[Unit]
Description=Run system_backup after mount
After=multi-user.target
RequiresMountsFor=/to/your/mount_path

[Service]
ExecStart=incremental_system_backup.sh
Type=oneshot
RemainAfterExit=false

[Install]
WantedBy=multi-user.target
```

`After=multi-user.target`
- **作用**: `After` 指定此服务在某个目标或服务之后启动。它只是设置启动顺序，并不意味着服务本身是依赖于目标。
- **示例**: `After=multi-user.target` 表示此服务会在系统达到 `multi-user.target` 状态之后启动。`multi-user.target` 表示系统已进入多用户模式，用户可以登录并使用系统。
- **解释**: 这意味着，只有在系统的多用户模式（即系统完成了基本的网络配置、登录等任务）启动之后，当前服务才会启动。

`RequiresMountsFor=/to/your/mount_path`
- **作用**: `RequiresMountsFor=` 是 `systemd` 提供的一个特殊配置项，用来明确指定某个挂载点的依赖关系。它确保服务的执行依赖于指定路径的挂载。
- **示例**: `RequiresMountsFor=/to/your/mount_path` 表示当前服务依赖于 `/to/your/mount_path` 路径的挂载。如果该路径没有被挂载，`systemd` 会阻止服务的启动，并在该挂载点完成挂载后再启动该服务。
- **解释**: 这意味着，如果指定的挂载点（如 `/to/your/mount_path`）没有被挂载，`systemd` 会等待直到该挂载点挂载完成，才能继续启动服务。这样可以防止服务在必要的挂载点未准备好时启动，避免出现错误。

### 设置服务自启

```bash
sudo systemctl enable system_backup_after_mount.service
```


## 设置服务定时执行

这部分是为我的用户数据做的，主要用到的是 [[K-12-定时器|定时器]] 和 [[K-13-Service（服务）|服务]]。

### 创建服务

在 `/etc/systemd/system/` 目录下创建一个 `service` 。比如：
```bash
sudo nano /etc/systemd/system/user_backup.service
```

```
# /etc/systemd/system/user_backup.service

[Unit]
Description=Run user_backup after mount
After=multi-user.target
RequiresMountsFor=/to/your/mount_path

[Service]
ExecStart=incremental_user_backup.sh
Type=oneshot
RemainAfterExit=false

[Install]
WantedBy=multi-user.target

```

### 创建定时器

在 `/etc/systemd/system/` 目录下创建一个 `timer` 。比如：
```bash
sudo nano /etc/systemd/system/user_backup.timer
```

```
# /etc/systemd/system/user_backup.timer

[Unit]
Description=Run user_backup weekly

[Timer]
OnCalendar=weekly
Persistent=true
Unit=user_backup.service

[Install]
WantedBy=timers.target
```

`OnCalendar=weekly`
- **作用**: 该字段定义了任务的调度时间。
- **示例**: `OnCalendar=weekly` 表示每周运行一次任务。
- **多种格式的示例**:
    - `OnCalendar=daily`：每天运行一次。
    - `OnCalendar=Mon *-*-* 12:00:00`：每周一中午12点运行。
    - `OnCalendar=*-01-01 00:00:00`：每年1月1日00:00运行。
    - `OnCalendar=2018-12-25 10:00:00`：在指定的日期和时间（2018年12月25日10:00）运行。

`Persistent=true`
- **作用**: 如果系统在预定时间未能运行该定时器（例如，系统在指定时间关闭或暂停），该字段指示 `systemd` 是否在下次系统启动时执行该定时器任务。
- **示例**: `Persistent=true` 表示即使系统在预定的时间关闭，定时器也会在系统启动时运行，确保任务不遗漏。
- **如果为 `false` 或缺省**: 系统不会执行错过的任务，只会在指定的下次时间运行。

`Unit=user_backup.service`
- **作用**: 这个字段指定该定时器触发的服务单元。它指示 `systemd` 在定时器触发时运行哪个具体的服务。
- **示例**: `Unit=user_backup.service` 表示定时器触发时会运行名为 `user_backup.service` 的服务单元。
- **常见用途**: 可以指定任何有效的服务文件，如 `backup.service`、`cleanup.service` 等。

`WantedBy=timers.target`
- **作用**: 该字段定义了该定时器启用时所属的目标。目标是 `systemd` 中的一种单位类型，用来组织和管理多个服务或定时器。
- **示例**: `WantedBy=timers.target` 表示该定时器属于 `timers.target`，即系统定时器目标，这通常用于启用所有定时器相关的功能。
- **其他目标**:
    - `WantedBy=multi-user.target`：意味着服务将在 `multi-user` 模式下运行。
    - `WantedBy=default.target`：将服务设置为默认启动。

### 设置定时器自动启动

```bash
sudo systemctl enable user_backup.timer
```

>[!tip] 服务不需要自启，它会被定时器叫醒的～
