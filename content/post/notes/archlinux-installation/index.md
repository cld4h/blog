---
title: Archlinux installation
date: 2019-09-24T15:31:24+0800
lastmod: 2022-02-20T16:38:30+0800
draft: false
---

<mark>
注意：本篇文章中提到的内容可能存在过时问题，另外有很多软件我已不再使用或反对使用，请小心！
</mark>

参考：
http://valleycat.org/linux/arch-usb.html

## 准备过程

* 下载iso

进入 https://www.archlinux.org/download/ 找到最近的镜像源下载iso即可

* 刷入U盘

[参见](https://wiki.archlinux.org/index.php/USB_flash_installation_media)

运行下述命令：

```sh
 dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

* 启动进入Live CD

### 调整Live archlinux root 分区的大小

[参见](https://www.ostechnix.com/adjust-size-root-partition-live-arch-linux/)

启动时按下 **e** 或 tab 键以修改内核参数

在 `linux=...initrd=...` 后添加 `cow_spacesize=1G`

如果没有在启动时修改内核参数，可以在启动后执行下述命令

```sh
mount -o remount,size=1G /run/archiso/cowspace
```

cow 是 Copy On Write 的缩写，额外的空间使用的是内存

## 开始安装（无桌面环境等）

### 验证系统启动模式：UEFI/BIOS

查看 `/sys/firmware/efi/efivars` 文件夹中的内容

```sh
ls /sys/firmware/efi/efivars
```
若有返回文件列表，则证明是在UEFI模式下
(或通过 `efivar -l` 命令也可查看)

### 连接网络

通过 `ip link` 查看连接情况

镜像默认启动时启动了 `dhcpcd` 服务

通过 `ping www.baidu.coom` 确认能上网

### 设置系统时间与NTP服务器同步

```sh
timedatectl set-ntp true
```

通过 `timedatectl status` 命令检查服务状态

### 磁盘分区

先 `lsblk` 一下确定磁盘名

通过 `gdisk /dev/sdX`  xzyy 全盘清空

通过 `cgdisk` 进行分区

下表中的分区方式支持BIOS和UEFI

| 编码 | 文件系统类型      | 分区大小     | 设备名(示例) |
|------|-------------------|--------------|--------------|
| ef02 | BIOS System       | 10MiB        | /dev/sda1    |
| ef00 | EFI System        | 500MiB       | /dev/sda2    |
| 8300 | Linux File System | 剩余全部空间 | /dev/sda3    |

swap 不设固定分区，由 swap 文件方式设置

格式化分区：

```sh
mkfs.fat -F32 /dev/sda2 #EFI System
mkfs.ext4 /dev/sda3 #Linux File System
```

不对 `/dev/sda1` 进行格式化

挂载分区：
```sh
mount /dev/sda3 /mnt
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```


### 设置镜像源

[支持IPv6的镜像源](https://www.archlinux.org/mirrorlist/?country=all&protocol=http&ip_version=6)
[镜像源列表生成器](https://www.archlinux.org/mirrorlist/)

编辑 `/etc/pacman.d/mirrorlist` 文件

创建和编辑 `/etc/pacman.d/mirrorlist` 文件之后，使用下面命令刷新镜像：

```sh
pacman -Syyu
```

传入两次 `--refresh` 或 `-y` 将强制更新所有软件包，即使系统认为它们已经是最新。

**每次修改镜像之后都应该使用**`pacman -Syyu`

[参考](https://wiki.archlinux.org/index.php/Mirrors)

### 安装软件

```sh
pacstrap -i /mnt base base-devel vim ntfs-3g networkmanager shadowsocks-libev
```

`-i` 选项表示"交互的(interactive)"

### 创建swap文件

我没有固定设置swap分区，而是使用了swap文件


```sh
sudo fallocate -l 1G /mnt/swapfile
```

If fallocate is not installed or if you get an error message saying fallocate failed: Operation not supported then you can use the following command to create the swap file:

```sh
sudo dd if=/dev/zero of=/mnt/swapfile bs=1024 count=1048576
```

设置swapfile文件权限

```sh
sudo chmod 600 /mnt/swapfile
```

```sh
sudo mkswap /mnt/swapfile
sudo swapon /mnt/swapfile
```

生成 `/etc/fstab`:
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

修改 `/etc/fstab` ，保证路径正确
```
/swapfile none swap defaults 0 0
```

验证 `swap` 已挂载

1. `sudo swapon --show`
2. `sudo free -h`

### chroot

```sh
arch-chroot /mnt
```

### 设置时区

```sh
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

运行 `hwclock` 以生成 `/etc/adjtime`
```sh
hwclock --systohc
```

### 本地化

编辑 `/etc/locale.gen` 文件，将 `en_US.UTF-8 UTF-8`  取消注释
之后运行 `locale-gen` 命令

新建 `/etc/locale.conf` 文件，将 `LANG` 变量设置为对应值

```
LANG=en_US.UTF-8
```

### 网络配置

编辑 `/etc/hostname` 文件，起一个好听好记的名字
如 `USB`

编辑 `/etc/hosts` 文件

```
127.0.0.1	localhost
::1		localhost
127.0.1.1	USB.localdomain	USB
```
### 针对于将系统安装在 portable USB key上

#### RAM disk image

[参见](http://valleycat.org/linux/arch-usb.html#RAM_disk)

(不用看this paragraph)
In order to boot the Linux Kernel persistently off of a USB device, some adjustments may be necessary to the initial RAM disk image. We need to ensure that block device support is properly loaded before any attempt at loading the filesystem. This not always the way a RAM disk image is configured in a generic Linux installation, and I suspect this may be another one of the failure points in other Linux USB installations out there. To configure a custom RAM disk image, open `/etc/mkinitcpio.conf` in an editor:

Ensure the **block** hook comes **before** the **filesystems** hook and **directly after** the **udev** hook like the following:

```
HOOKS=(base udev block filesystems keyboard fsck)
```

 Now regenerate the initial RAM disk image with the changes made:

```sh
mkinitcpio -p linux
```

#### network interface names

 Arch Linux's basic service manager, systemd, assigns network interfaces predictable names based on the actual device hardware. This is great for just about any other type of install, but can pose some problems for the portable USB installation we're going for. To ensure that the ethernet and wifi interfaces will always be respectively named _eth0_ and _wlan0_, revert the Arch Linux USB back to traditional device naming:

```sh
ln -s /dev/null /etc/udev/rules.d/80-net-setup-link.rules
```

####  journal config

A default installation of Arch Linux is setup with systemd to continuously journal various information about current processes and write that data to storage on disk. For a persistent bootable installation on a flash memory device, however, we can change some options in `journald.conf` to enable journal keeping entirely in RAM (thus reducing writes to the flash device). To control where journal data is stored, open `/etc/systemd/journald.conf` in and editor, to switch journal data storage to RAM, set the storage variable to volatile by ensuring the following line is uncommented:

```
Storage=volatile
```

As an additional precaution, to ensure the operating system doesn't overfill RAM with journal data, set the max-use variable by adding the line:

```
SystemMaxUse=16M
```

####  mount options

Modern filesystems are able to record various metadata (last accessed, last modified, user rights, etc.) about their files. A default filesystem mount generally keeps track of as much as this information as possible. For a persistent bootable operating system on a flash memory device, however, we should limit some of this record keeping in order to reduce writes to the flash device. Using the `noatime` mount option in fstab will disable the record keeping of file access times: no writes will occur when a file is read, only when it is modified.

To disable record keeping of file access times for the bootable USB, open /etc/fstab in an editor, change the mount options from `relatime` to `noatime`.

### Root 密码

通过 `passwd` 修改root密码

### Boot loader

参考：
[BIOS and UEFI](http://valleycat.org/linux/arch-usb.html#bootloader)

1. 确保 `grub` 和 `efibootmgr` 已安装
2. 确保 EFI 系统分区已经挂载（前面通过 `mount /dev/sda2 /mnt/boot` 挂载，对于此时 (chroot 之后) 来说，挂载点为 `/boot`），
3. 运行

同时安装两个启动项 [BIOS and UEFI](http://valleycat.org/linux/arch-usb.html#bootloader)
Setup GRUB for MBR/BIOS booting mode:
```sh
grub-install --target=i386-pc --boot-directory /boot /dev/sda
```
Setup GRUB for UEFI booting mode:
```sh
grub-install --target=x86_64-efi --efi-directory /boot --boot-directory /boot --removable
```

[GRUB](https://wiki.archlinux.org/index.php/GRUB#Installation_2) wiki中推荐的安装指令(与前面UEFI安装类似)
```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

4. 可以安装 `os-prober` 软件包并运行来检测其他OS
```sh
sudo pacman -S os-prober
os-prober
```
5. 生成 `grub.cfg` 配置文件
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### 网络

Install the ifplugd package to configure automatic IP leasing on ethernet devices:

```sh
pacman -S ifplugd
```

Download the packages to enable wifi support with a basic command line interface:

```sh
pacman -S iw wpa_supplicant dialog
```

安装 NetworkManager 并设置开机启动

```sh
sudo pacman -S networkmanager
sudo systemctl enable NetworkManager
```

### 显卡

To support most common GPUs, install all four basic open source video drivers:

```sh
pacman -S xf86-video-ati xf86-video-intel xf86-video-nouveau xf86-video-vesa
```

### 触摸板

Install support for standard notebook touchpads:

```sh
pacman -S xf86-input-synaptics
```
在 `.xinitrc` 文件中 `exec` 前添加
```sh
synclient TapButton1=1
synclient TapButton2=3
synclient TapButton3=2
synclient VertScrollDelta=-111
```

### 电池

Install support for checking battery charge and state:

```sh
pacman -S acpi
```

### 针对于SSD的TRIM指令的优化

将持续TRIM改为定时TRIM
```sh
sudo systemctl enable fstrim.timer
```
参见
https://www.digitalocean.com/community/tutorials/how-to-configure-periodic-trim-for-ssd-storage-on-linux-servers

### 创建用户

```sh
useradd -m -G wheel,storage hao
# useradd -m -g users -G wheel,storage -s /bin/bash hao
```

`-m` 选项创建home目录
`-g` 选项指定所在组
`-G` 选项指定包含所创建用户的额外的组
`-s` 选项指定用户所用的shell

运行上面的没有注释的命令会创建一个用户 `hao`(UID=1000) 和一个组 `hao`(GID=1000)

我们想把组 `hao`(GID=1000) 删除，把用户 `hao`(UID=1000) 更改到组 `users` 下

```sh
usermod -g users -G wheel,storage,power hao
groupdel hao
```

其他的可用命令 `useradd`,`userdel`,`groupadd`,`groupdel`

#### 编辑 `/etc/sudoers`

```sh
EDITOR=vim visudo
```

```sudoers
##
## User privilege specification
##
root ALL=(ALL) ALL

## Uncomment to allow members of group wheel to exec
ute any command
# %wheel ALL=(ALL) ALL

## Same thing without a password
# %wheel ALL=(ALL) NOPASSWD: ALL
%wheel ALL=(ALL) NOPASSWD: ALL

## This keeps you from needing to reinsert your pass
words in each different terminal a wheel user uses s
udo in
Defaults !tty_tickets
```

## 安装图形界面

### 安装 X.org (X)

```sh
pacman -S xorg-server xorg-xinit
```

输入 `xinit` 或 `startx` 后， X Server 会读取 `$HOME/.xinitrc` 来查找如何启动
如，要启动 i3，则在`$HOME/.xinitrc`中添加 `exec i3`

### 安装 i3 等相关程序

```sh
pacman -S i3 dmenu ttf-dejavu firefox alacritty ranger
```

### 中文字体

环境变量 `LANG=zh_CN.UTF-8`

* `.bashrc`：每次**终端登录**时读取并运用里面的设置。
* `.xinitrc`：每次`startx` **启动X界面**时读取并运用里面的设置。
* `.xprofile`：每次使用**gdm等图形登录**时读取并运用里面的设置。

在 `.xinitrc` 或 `.xprofile`中添加下面三行内容
```
export LANG=zh_CN.UTF-8
export LANGUAGE=zh_CN:en_US
export LC_CTYPE=en_US.UTF-8
```

安装思源字体
```sh
pacman -S adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts
```

### 中文输入法---~~搜狗拼音输入法~~
<mark>强烈反对使用搜狗拼音输入法，建议使用自由的rime输入法。前者不但可能存在后门；配置时还有很多坑。</mark>

```sh
sudo pacman -S fcitx fcitx-im fcitx-configtool
```

由于搜狗拼音输入法不在官方仓库中，我们可以用AUR，也可以添加非官方仓库[archlinuxcn]

编辑 `/etc/pacman.conf`，在所有仓库之后加入如下内容
```
[archlinuxcn]
#SigLevel = Never
Include = /etc/pacman.d/archlinuxcn.mirrorlist
```

编辑 `/etc/pacman.d/archlinuxcn.mirrorlist` 文件
加入如下内容
```
## 清华大学 (ipv4, ipv6, http, https)
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch

## 中国科学技术大学 (ipv4, ipv6, http, https)
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch

## Our main server (ipv4, ipv6, http, https)
## Our main server located in Netherlands
Server = https://repo.archlinuxcn.org/$arch

## 浙江大学 (浙江杭州) (ipv4, ipv6, http, https)
## Added: 2017-06-05
Server = https://mirrors.zju.edu.cn/archlinuxcn/$arch
```

运行 `sudo pacman -Syyu` 和 `sudo pacman -S archlinuxcn-keyring`
> pacman 清除缓存
> `sudo pacman -Sc`

[更多源参见](https://github.com/archlinuxcn/mirrorlist-repo)

~~安装搜狗拼音输入法~~

<mark>！请不要安装搜狗拼音输入法！</mark>

```
# sudo pacman -S fcitx-sogoupinyin
sudo pacman -S fcitx-rime
```

在 `~/.profile` 文件中加入下面几个环境变量：
```sh
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```
在 `.xinitrc` 中添加
```sh
fcitx &
```
或在 i3配置文件中添加 `exec_always fcitx`

### 安装 virtualbox

```
sudo pacman -S linux-headers virtualbox virtualbox-guest-iso
```

### 修改键位

#### 使用 `xmodmap`

```sh
xmodmap -pke > ~/.xmodmap
```

`xev` 可以查看keycode

编辑上一步得到的 `~/.xmodmap`文件，对应的keycode作修改
在文件开始处添加
```
clear control
clear lock
clear mod1
```
在文件结尾处添加
```
add control =Control_L Control_R
add mod1 = Alt_L
```

执行 xmodmap `~/.xmodmap`


### 安装无道词典

<mark>
我现在使用的是另一个程序：<a href="https://github.com/TimothyYe/ydict">ydict</a>
</mark>

```sh
sudo pacman -S python python-pip
sudo pip install bs4
sudo pip install lxml
```

**注意，无道词典与variety冲突时，只需要pip uninstall 冲突的包即可**

### 配置neovim

将`.config/nvim/init.vim` 软链接到 `~/.vimrc`
编辑`~/.vimrc`
使用`vim-plug`工具管理vim插件
在vim中通过`:PlugInstall`安装插件[参考](https://github.com/junegunn/vim-plug/wiki/tutorial#installing-plugins)

#### UltiSnips 需要python支持
由于UltiSnips需要python支持，当启动neovim时，若没有pynvim包，则会报错：
> UltiSnips: the Python version from g:UltiSnipsUsePythonVersion (3) is not available.

解决方案：pip安装pynvim
```sh
sudo pip install -U pynvim
```

### 美化主题

```
# for GTK
sudo pacman -S lxappearance
# for qt5
sudo pacman -S qt5ct
```

更换桌面壁纸
```
sudo pacman -S feh variety
```

终端半透明
```
sudo pacman -S compton
```

alacritty: `.config/alacritty/config.yml` 设置 `opacity`
