---
title: "Making a bootable MacOS X USB on Archlinux"
date: 2023-05-10T14:05:43+0800
draft: false
---

A few days ago, I got a MacBook Air (mid 2011) from a friend.
The battery was died, but everything else is fine, despite it running extremly slowly.

I tried to reinstall an older version of OS X system for it.

The first thing is to create a bootable USB drive for that. Following the instructions [here](https://bramblecentral.wordpress.com/2020/05/19/making-a-bootable-macos-x-usb-on-linux/) and [here](https://github.com/eprigorodov/mkosxinstallusb), I've done this successfully.

It's a completely a new (and a little complicated) experience for me. So I felt it's necessary to write up the steps.

## Preparation

* `dmg2img` (in AUR) to convert Apple's `.dmg` image format to `.img`
* `kpartx` (in `multipath-tools` package) to mount `.img`
* `xar` (in AUR) to extract `.pkg`
* `InstallMacOSX.dmg` or `InstallESD.dmg`
* A USB drive (refered to as `/dev/sdX` in the following contents)

## Steps

1. Extract `InstallESD.dmg` from `InstallMacOSX.dmg`
2. Do the following steps:
```sh
mkdir -p /mnt/OSX_InstallESD /mnt/OSX_BaseSystem /mnt/usbstick

# convert installer disk image to raw format
dmg2img "Install OS X <Version>.app/Contents/SharedSupport/InstallESD.dmg" InstallESD.img
kpartx -a InstallESD.img
mount /dev/mapper/loop0p2 /mnt/OSX_InstallESD

# convert base system disk image to raw format
dmg2img /mnt/OSX_InstallESD/BaseSystem.dmg BaseSystem.img
kpartx -a BaseSystem.img
mount /dev/mapper/loop1p1 /mnt/OSX_BaseSystem

# partition the USB flash drive, /dev/sdX
sgdisk -o /dev/sdX
sgdisk -n 1:0:0 -t 1:AF00 -c 1:"disk image" -A 1:set:2 /dev/sdX
mkfs.hfsplus -v "OS X Base System" /dev/sdX1
mount /dev/sdX1 /mnt/usbstick

# copy installer files
rsync -aAEHW /mnt/OSX_BaseSystem/ /mnt/usbstick/
rm -f /mnt/usbstick/System/Installation/Packages
rsync -aAEHW /mnt/OSX_InstallESD/Packages /mnt/usbstick/System/Installation/
rsync -aAEHW /mnt/OSX_InstallESD/BaseSystem.chunklist /mnt/usbstick/
rsync -aAEHW /mnt/OSX_InstallESD/BaseSystem.dmg /mnt/usbstick/
sync
```

In the final step, `sync` is to write all cached writes to persistent storage. It can go extremly slowly. To see the progress, you can look at `/proc/meminfo` to see `Dirty` sizes.

```sh
watch -d grep -e Dirty: -e Writeback: /proc/meminfo
```

[(reference)](https://unix.stackexchange.com/questions/48235/can-i-watch-the-progress-of-a-sync-operation)
