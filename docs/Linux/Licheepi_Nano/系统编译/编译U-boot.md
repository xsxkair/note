---
title: 编译U-BOOT
tags: teedoc, markdown, 语法
keywords: teedoc, markdown, 语法
desc: teedoc 的 markdown 语法介绍和实例
date: 2023-02-01 14:17:13
permalink: /pages/licheepi_nano/build_system/u-boot/
---



> 下载荔枝派u-boot

```
git clone https://gitee.com/LicheePiNano/u-boot.git
cd u-boot

# 查看分支
git branch -a
# 切换到 Nano 分支
git checkout nano-lcd800480
```

> 去掉官方写死在文件中的bootcmd，此步骤可忽略，因为目标也是编译出针对flash的u-boot，只是为了通用，将写死的配置去掉，在menuconfig中配置

```
include/configs/suniv.h

注释掉以下内容
#define CONFIG_BOOTCOMMAND   "sf probe 0 50000000; "                    \
                     "sf read 0x80C00000 0x100000 0x4000; "  \
                     "sf read 0x80008000 0x110000 0x400000; " \
                     "bootz 0x80008000 - 0x80C00000"
```

> 加载荔枝派官方的flash配置文件

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- licheepi_nano_spiflash_defconfig
```

> 打开可视化配置界面，配置bootcmd 和 bootarg

```
make ARCH=arm menuconfig

#root=/dev/mmcblk0p1为文件系统的路径，设置为TF卡
#TF卡插入后，
[*] Enable boot arguments
	console=ttyS0,115200 panic=5 rootwait root=/dev/mmcblk0p1 earlyprintk rw
	
#分别设置从flash的哪个位置加载dtb和zimage
[*] Enable a default value for bootcmd
	sf probe 0 50000000;sf read 0x80C00000 0x100000 0x4000;sf read 0x80008000 0x110000 0x400000;bootz 0x80008000 - 0x80C00000
```

> 开始编译

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j8
```

> 烧录u-boot

```
sunxi-fel -p spiflash-write 0 ./u-boot-sunxi-with-spl.bin
```

