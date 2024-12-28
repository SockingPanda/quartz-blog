---
title: 基于“Duplicity”的增量备份
date: 2024-12-02 02:14
tags:
  - 实验记录
status: 已完成
public: true
---
>[!failure] 上古老工具，链式增量备份，没办法删除前面的增量记录

>[!danger] 当google 第一页发现2013年的文章的时候，我就知道自己被 GPT 骗了（但还是去尝试了一番，了解到了 GNG加密工具） 

Duplicity 是一个支持增量备份的工具，可以将备份存储到 NTFS，同时压缩和加密数据。

### **备份步骤**

1. **安装 Duplicity**
```bash
sudo apt install duplicity
```

2. **执行增量备份**
```bash
sudo duplicity backup --allow-source-mismatch \
  --exclude /dev \
  --exclude /proc \
  --exclude /sys \
  --exclude /tmp \
  --exclude /run \
  --exclude /mnt \
  --exclude /media \
  --exclude /lost+found \
  --no-encryption \
  / \
  "file://<目标位置>"
```

**排除目录的解释**：

1. **`/dev`**：这个目录包含了设备文件，是 Linux 系统用来与硬件交互的接口。设备文件并不是常规文件，它们代表硬件设备（如磁盘、终端、内存等）。因此，`/dev` 目录不需要备份。
    
2. **`/proc`**：这个目录是一个虚拟文件系统，包含了关于系统和进程的实时信息（例如 CPU、内存、设备状态等）。这些文件是动态生成的，并不是常规存储的文件，因此无需备份。
    
3. **`/sys`**：`/sys` 也是一个虚拟文件系统，提供了有关内核、设备驱动程序和硬件的实时信息。它不包含常规数据，因此也不需要备份。
    
4. **`/tmp`**：这个目录用于存放临时文件，在系统运行时生成，系统重启后通常会被清空。因为这些文件不持久化，所以也不需要备份。
    
5. **`/run`**：这个目录包含了运行时数据（如正在运行的服务的 PID 文件、套接字等），这些数据在系统重启时会丢失，因此不需要备份。
    
6. **`/mnt`**：通常这个目录用于挂载外部存储设备（如硬盘、U 盘等）。`duplicity` 默认排除它，因为这些存储设备上的数据通常不需要备份，或者你可能希望将其视为单独的备份目标。
    
7. **`/media`**：这个目录用于挂载可移动存储设备（如 USB 驱动器、光盘等）。和 `/mnt` 一样，通常你不需要备份这些外部存储设备中的内容。
    
8. **`/lost+found`**：这个目录通常存在于文件系统的根目录下，用于存放被系统错误或文件系统损坏时恢复的文件。这个目录的内容也是动态生成的，通常不需要备份。

### GPG加密
Duplicity 内置 [[K-10-GPG加密|GPG加密]] ，首先使用它来生成新的密钥对：
```bash
gpg --full-generate-key
```
然后跟着提示一路往下就对了～下面是我的过程（隐去敏感信息）：
```bash
gpg (GnuPG) 2.4.4; Copyright (C) 2024 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

请选择您要使用的密钥类型：
   (1) RSA 和 RSA 
   (2) DSA 和 Elgamal 
   (3) DSA（仅用于签名）
   (4) RSA（仅用于签名）
   (9) ECC（签名和加密） *默认*
  (10) ECC（仅用于签名）
  （14）卡中现有密钥 
您的选择是？ 
```

```bash
请选择您想要使用的椭圆曲线：
   (1) Curve 25519 *默认*
   (4) NIST P-384
   (6) Brainpool P-256
您的选择是？ 
```

```bash
请设定这个密钥的有效期限。
         0 = 密钥永不过期
      <n>  = 密钥在 n 天后过期
      <n>w = 密钥在 n 周后过期
      <n>m = 密钥在 n 月后过期
      <n>y = 密钥在 n 年后过期
密钥的有效期限是？(0) 
密钥永远不会过期
这些内容正确吗？ (y/N) y
```

```bash
GnuPG 需要构建用户标识以辨认您的密钥。

真实姓名： name
电子邮件地址：mail 
注释： 
您选定了此用户标识：
    “name <mail>”

更改姓名（N）、注释（C）、电子邮件地址（E）或确定（O）/退出（Q）？ O
我们需要生成大量的随机字节。在质数生成期间做些其他操作（敲打键盘
、移动鼠标、读写硬盘之类的）将会是一个不错的主意；这会让随机数
发生器有更好的机会获得足够的熵。
我们需要生成大量的随机字节。在质数生成期间做些其他操作（敲打键盘
、移动鼠标、读写硬盘之类的）将会是一个不错的主意；这会让随机数
发生器有更好的机会获得足够的熵。
gpg: 目录‘/home/your_user/.gnupg/openpgp-revocs.d’已创建
gpg: 吊销证书已被存储为‘/home/your_user/.gnupg/openpgp-revocs.d/**.rev’
公钥和私钥已经生成并被签名。
```

输入 `gpg --list-secret-keys --keyid-format LONG` 查看密钥ID，输出大概是下面这样的：
```bash
/home/your_user/.gnupg/secring.gpg
------------------------------
sec   4096R/<YOUR_KEY_ID> 2019-12-12 [expires: 2021-12-12]
uid                          Your Name <you@example.com>
ssb   4096R/<YOUR_SUBKEY_ID> 2019-12-12
```
`
### 执行加密备份

上次系统搞崩溃用户数据其实是没有任何影响的，所以我的备份策略是：系统每天备份、用户数据每周备份。

```bash
sudo PASSPHRASE=$(sudo cat ~/.gnupg/passphrase.txt) duplicity backup --gpg-options "--keyring /home/your_user/.gnupg/pubring.kbx" --skip-if-no-change /etc file:///path/to/etc

sudo PASSPHRASE=$(sudo cat ~/.gnupg/passphrase.txt) duplicity backup --gpg-options "--keyring /home/your_user/.gnupg/pubring.kbx" --skip-if-no-change /bin file:///path/to/bin

sudo PASSPHRASE=$(sudo cat ~/.gnupg/passphrase.txt) duplicity backup --gpg-options "--keyring /home/your_user/.gnupg/pubring.kbx" --skip-if-no-change /lib file:///path/to/lib

sudo PASSPHRASE=$(sudo cat ~/.gnupg/passphrase.txt) duplicity backup --gpg-options "--keyring /home/your_user/.gnupg/pubring.kbx" --skip-if-no-change /usr file:///path/to/usr

sudo PASSPHRASE=$(sudo cat ~/.gnupg/passphrase.txt) duplicity backup --gpg-options "--keyring /home/your_user/.gnupg/pubring.kbx" --skip-if-no-change /var file:///path/to/var
```

运行的时候依然需要每次输入 `GPG 密码短语` 。为了能够结合自动化备份功能，我们将密码短语存储在某一个文件中，然后让 `duplicity` 使用 `--gpg-options` 选项来指定密码文件。

```bash
duplicity backup --gpg-options "--passphrase-file /path/to/passphrase_file" <source> <destination>
```


---

### **恢复步骤**

1. **恢复特定时间点的数据**
    
	```bash
	duplicity restore --time <版本> "file://<备份位置>" <要恢复的目录>
	```
	
	比如：
	```bash
	duplicity restore --time 2024-11-20T00:00:00 "file://<备份位置>" /
	```
    

---

不行了不行了，tmd这个太复杂了……实在是接受不了这个增量保存记录没办法删除的鬼畜玩意儿……他这个是，后面的记录是基于前面的记录做修改的，那么问题来了，假设我不需要3次以前的增量记录了呢？我还删除不了，貌似只能恢复、重新备份？拿不是太麻烦了……（而且资料也不多）

