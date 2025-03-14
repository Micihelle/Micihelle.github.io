---
layout: post
title: Arch-recovery
date: '2025-03-14 11:37:07 +0800'
description: 安装windows+Arch双系统之后的一次恢复操作
---



<!-- 我是一个Arch Newbie，我在某一次系统黑屏后，尝试用U盘进入chroot环境恢复了系统并总结了这样的博文，你能否帮我分析一下我在本次操作中比较幼稚和不够专业的地方，并给我提出在文本方面和技术方面的改进？--!>

<!--我对我的博文进行了一定的修改，你能否帮我分析一下我在本次操作中比较幼稚和不够专业的地方，并给我提出在文本方面和技术方面的改进？!-->


## 故障背景
- 问题描述：KDE 中文设置导致黑屏
- 恢复工具：Arch Live USB

在安装桌面环境KDE后，没有设置好中文支持，然后更改语言设置直接导致了系统黑屏（只能看到一个鼠标），查阅了一下资料+Deepseek之后，尝试用此前安装系统时准备的U盘恢复Arch系统。


## 恢复流程
### （1） Live CD 环境准备
准备u盘进入到live CD环境，然后配置网络

> info && ai agent
> 为什么Live CD配置的网络，能够在croot环境下自动继承呢？
- 仅文件系统被隔离：你在 chroot 环境中操作的是目标系统的文件，但网络、进程管理等仍由宿主系统控制。

```
Live CD 内核 → 管理网络栈
  |
  ├─ 物理网卡驱动
  └─ chroot 环境 → 共享内核网络接口（通过 /proc 和 /sys）
```


<!--我是一名Arch Newbie，前面安装双系统的时候一直不太理解进入chroot到底是什么意思，没想到原来可以直接用U盘进入live cd环境进行系统恢复为什么Live CD配置的网络，能够在croot环境下自动继承呢？!-->


### （2）网络配置
``` bash 
iwctl  # 进入交互式无线管理
station list  # 列出无线设备
station wlan0 scan  # 扫描网络
station wlan0 connect SSID  # 连接 Wi-Fi（替换为你的 SSID）
exit

# 测试网络连通性
ping -c 4 8.8.8.8
```

### （3）挂载与 arch-chroot
在第一次安装系统的时候已经创建btrfs子卷了，所以直接挂载物理分区

``` bash 
lsblk #检查磁盘分区情况（将后面提到的<root_partition>等替换成具体设备
mount -t btrfs -o compress=zstd,subvol=@ <root_partition> /mnt #挂载根目录
mount -t btrfs -o compress=zstd,subvol=@home <root_partition> /mnt/home --mkdir #挂载home目录
mount -t btrfs -o compress=zstd,subvol=@swap <root_partition> /mnt/swap --mkdir #可选，挂载未来可能用到的swap文件
mount <efi partition> /mnt/efi --mkdir #挂载efi分区,我这里是/dev/nvme0n1p1
mount <boot partition> /mnt/boot --mkdir #挂载/boot分区
swapon <swap partition> #启用swap分区

#验证挂载情况
lsblk #复查硬盘分区挂载情况
free -h #复查swap分区挂载情况

arch-chroot #arch-chroot系统
fish 
pacman -Syyu
```




### （4）卸载KDE
``` bash 
sudo pacman -Rns plasma kde-applications console # 卸载 KDE Plasma 和官方应用套件
sudo pacman -Rns sddm                    # 卸载 KDE 默认的显示管理器（SDDM）

# 删除未被其他软件依赖的孤立包（可选但推荐）
sudo pacman -Rns $(pacman -Qdtq)

# 检查配置
ls ~/.config/plasma*      # KDE Plasma 配置
ls ~/.local/share/plasma* # 用户数据
ls ~/.kde                 # 旧版 KDE 配置目录
ls ~/.cache/*             # 清理缓存（谨慎操作，可能影响其他程序）

sudo ls /etc/sddm.conf    # SDDM 配置文件
sudo ls /usr/share/sddm   # SDDM 主题和资源

~/.config/sddm/*    # 用户级 SDDM 配置（可能残留错误主题设置）
```



### （5）Localization

添加中文字体：
``` bash
sudo pacman -S noto-fonts-cjk adobe-source-han-sans-cn-fonts
sudo fc-cache -fv
sudo pacman -S fcitx5 fcitx5-chinese-addons
sudo systemctl status gdm    # 或 lightdm/sddm
sudo ls /etc/X11/xorg.conf  # 检查旧配置是否清除
```

添加中文支持：
``` bash
locale  # 查看当前生效的 locale 设置
cat /etc/locale.gen | grep zh_CN.UTF-8  # 确认是否已取消注释

ls /usr/share/fonts/  # 检查是否存在 noto-cjk 或 adobe-source-han 字体
fc-list | grep "Noto Sans CJK"  # 验证字体是否被系统识别
```

### （6）安装GNOME桌面环境
``` bash
# 安装 GNOME 核心组件
sudo pacman -S gnome gnome-extra
# 安装显示管理器 GDM
sudo pacman -S gdm
# 启用 GDM 服务
sudo systemctl enable gdm
```

### （7）显卡驱动验证
``` bash

lspci | grep VGA        # 查看显卡型号
pacman -Q linux-firmware xf86-video-*  # 检查驱动包

reboot
```
然后可以看到桌面正常显示了。


## 结束语
<!--有一说一，fish功能用的也太爽了，还能记录某一环境下的历史指令 然后可以很方便的补全，比第一次安装系统的时候舒服多了（!-->


- [arch-chroot](https://wiki.archlinux.org/title/Chroot#Using_arch-chroot)