---
title: 折腾记：给 PVE 虚拟机缩盘，踩了三个坑
published: 2026-04-21
description: 一次本该简单的缩盘操作，最后踩了三个坑才搞定。记录下来，给自己也给同样在折腾的人。
tags: [PVE, homelab, Linux, 虚拟机]
category: tinkering
draft: false
---

家里的 PVE 跑着几台虚拟机，其中一台 Ubuntu 分配了 256G 的系统盘。

实际用了多少？不到 30G。

这种事放着就很难受。thin pool 的空间就那么多，这台机子一个人占着一大块，却只用了零头。于是我决定给它缩盘，目标是 100G，留足缓冲，也不至于太浪费。

以为是件简单的事。结果折腾了两天。

---

## 第一次：原地缩盘，翻车

最开始的思路很直觉：挂一张 GParted Live，把根分区从 255G 缩到 78G，再在 PVE 宿主把底层的 LVM thin volume 收小，完事。

步骤看起来没问题，实际上有个我当时没意识到的陷阱——

**GPT 磁盘的备份头在磁盘尾部。**

GParted 把分区缩完之后，GPT 的主头和备份头还是按原来 256G 的位置放的。这时候你在 PVE 宿主用 `lvreduce` 把底层 LV 从 256G 直接截到 80G，相当于把备份头那块直接切掉了。

结果很惨：VM 重启，直接掉 initramfs，`/dev/sda` 找不到分区。

我重新进 GParted Live 去看，`/dev/sda` 显示整盘 unallocated，分区表已经损坏。

好在之前做了快照，回滚回去，什么都没丢。但这条路明显走不通了，需要换个方向。

---

## 第二次：新盘迁移法

换了个思路：不缩旧盘，而是新建一块 100G 的盘，把数据迁移过去，修好引导，切换启动盘。旧盘留着，观察几天没问题再删。

这个方案最大的好处是——**每一步都有退路**。旧盘全程不动，切换失败了直接改回来，没有不可逆的操作。

执行过程大体上是：

1. PVE 里关机 VM，建 100G 新盘挂到 scsi1
2. 挂 GParted Live ISO，从 ISO 启动
3. 在 live 环境里给新盘建分区（GPT + bios_grub + EFI + ext4 根）
4. rsync 把旧盘数据完整迁到新盘
5. 修 fstab，把 UUID 换成新盘的
6. 重新装 grub 引导到新盘
7. 改 PVE 启动顺序，切到新盘启动
8. 验证

听起来清晰，实际踩了三个坑。

---

## 三个坑

### 坑一：GParted Live 里 SSH 进不去

因为 noVNC 控制台没法粘贴命令，我想从外部 SSH 进 GParted Live 接管操作。结果发现进不去——

默认配置密码登录关着，`/etc/hosts.deny` 里还写了一行 `ALL: ALL EXCEPT localhost`，把所有外部连接全拒了。

改掉这两处就好：

```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
sudo sed -i '/^ALL: ALL EXCEPT localhost/d' /etc/hosts.deny
```

另外每次重启 GParted Live 都会生成新的 host key，之前 known_hosts 里的记录会冲突，记得清掉：

```bash
ssh-keygen -f ~/.ssh/known_hosts -R '<your-vm-ip>'
```

### 坑二：SeaBIOS 还是 UEFI，要先搞清楚

旧盘上有 EFI 分区，我想当然以为这台 VM 是 UEFI 启动的，于是装 grub 用了 `--target=x86_64-efi`。

失败了。

后来才发现 PVE 的 VM 配置里根本没有 `bios: ovmf` 这行——说明用的是 SeaBIOS，传统 BIOS 模式。

**判断方法就一条：看 VM 配置里有没有 `bios: ovmf`。有才是 UEFI，没有就是传统 BIOS。**

旧盘上有 EFI 分区是怎么回事？大概是 Ubuntu 安装时建了，但 VM 实际上是用 legacy grub + bios_grub 分区启动的，EFI 那块没有被真正用到。

改成 `--target=i386-pc` 装 legacy grub。但又遇到下一个问题——

### 坑三：chroot 内 grub-install 报 unknown filesystem

在 chroot 环境里跑 `grub-install --target=i386-pc /dev/sdb`，一直报 `unknown filesystem`。加 `--skip-fs-probe` 也没用。

折腾了一会儿，换了个思路：**从 chroot 外直接装，用 `--boot-directory` 指定新盘的 boot 目录**：

```bash
sudo grub-install --target=i386-pc --skip-fs-probe \
  --boot-directory=/mnt/new/boot /dev/sdb
```

这样绕过了 chroot 里的探测问题，安装成功。

---

## 还没完：启动掉 initramfs

改完引导，切换新盘启动，还是掉了 initramfs。

错误信息是 `FEATURE_C12 unsupported`。

查了一下，原来新版 `mkfs.ext4` 默认会启用一个叫 `orphan_file` 的特性，内核标识是 `FEATURE_C12`。但这台 Ubuntu 系统内的 `e2fsck` 版本是 2021 年的 1.46.5，不认识这个特性，启动时 fsck 一跑就报错退出，然后掉进 initramfs。

临时解决：在 initramfs 提示符里手动挂载跳过 fsck：

```
mount -o noload /dev/sdb3 /root
exec switch_root /root /sbin/init
```

进了系统之后，永久修复——在 grub 参数里加 `fsck.mode=skip`：

```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"/GRUB_CMDLINE_LINUX_DEFAULT="quiet splash fsck.mode=skip"/' /etc/default/grub
sudo update-grub
```

重启，自动起来，正常。

---

## 最后的状态

系统盘从 256G 缩到了 100G，已用 29G，剩 64G。Docker 上跑的六个服务全部正常。旧盘还挂着，观察几天没问题再删。

这次折腾前后用了快两天，其实核心操作不算复杂，主要时间都花在排查那三个坑上。

回头看，每一个坑其实都有迹可循——

- GParted Live 的 SSH 限制是有文档的，只是懒得查
- SeaBIOS 还是 UEFI 应该是第一步就确认的事，不该靠猜
- mkfs.ext4 的版本兼容问题，多想一步就能预判到

下次遇到类似操作，应该会顺很多。

---

## 术语补充：给和我一样的小白看的

### 1. 什么是虚拟机系统盘？
可以把它理解成，一台虚拟电脑自己的硬盘。虽然它跑在 PVE 这种虚拟化平台里，但对虚拟机里的 Ubuntu 来说，这块盘就是它真正拿来装系统、存文件的地方。

### 2. 什么是根盘 / 根分区？
就是系统真正“住”的地方，通常挂载在 `/`。你可以粗暴理解成：这是整台 Linux 最核心的区域。动普通数据盘，和动根盘，风险完全不是一个级别。

### 3. GParted Live 是什么？
它不是你原来系统的一部分，而是一个临时启动的维护系统。因为你要缩的是正在使用的系统盘，所以不能站在地板上锯地板，必须先从另一个系统启动，再去动原来的盘。

### 4. 什么是 thin pool？
我一开始也不懂。后来简单理解成：PVE 底下有一个大池子，虚拟机的磁盘空间都是从这个池子里“账面分配”出去的。表面看每台机器分了很多盘，但底层真正占了多少，要看实际写入的数据。

### 5. 为什么原地缩盘会翻车？
因为这不是改一个数字那么简单。你看到的“盘变小了”，背后其实牵涉文件系统、分区表、虚拟磁盘边界这些东西。顺序一错，就可能不是“缩小一点”，而是直接把启动搞坏。

### 6. 什么是分区表？
可以把它理解成硬盘里的“目录索引”。系统要靠它知道这块盘从哪里开始、哪里结束、哪一段装的是什么。如果这份索引坏了，系统就会变成“我知道这里有盘，但我不知道里面原来的结构在哪”。

### 7. 什么是引导 / grub？
电脑或者虚拟机开机时，不是直接进系统的，中间要先经过一段“把系统拉起来”的流程。`grub` 就是这里面的关键角色之一。系统文件就算都还在，只要引导坏了，照样可能起不来。

### 8. initramfs 又是什么？
可以把它理解成 Linux 正式启动前的一个“过渡维修小环境”。如果系统找不到根盘、或者检查文件系统时出问题，常常就会掉进这里。它不是系统彻底没了，而是启动流程卡在半路了。

### 9. 为什么后来改成“新盘迁移法”？
因为这条路更稳。不是在原来的盘上硬砍，而是新建一块更小的盘，把系统搬过去。这样旧盘全程还在，出问题了还能退。说白了，**这是一种更保守、更适合小白的做法。**

### 10. 这次真正学到的是什么？
不是某一条命令，也不是记住了多少术语，而是：

**系统层的操作，最重要的不是“敢不敢动”，而是“有没有退路”。**

<div style="text-align: right;">
2026.04.21<br>
上海
</div>
