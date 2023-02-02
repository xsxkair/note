---
title: 编译Linux
tags: teedoc, markdown, 语法
keywords: teedoc, markdown, 语法
desc: teedoc 的 markdown 语法介绍和实例
date: 2023-02-01 14:17:13
permalink: /pages/licheepi_nano/build_system/linux/
---


> 下载，编译

```
git clone https://gitee.com/LicheePiNano/Linux.git

make ARCH=arm f1c100s_nano_linux_defconfig

make ARCH=arm menuconfig

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j8

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16 INSTALL_MOD_PATH=out modules

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16 INSTALL_MOD_PATH=out modules_install
```

> 烧录

u-boot的bootcmd中，写入内容如下

```
#类似连接flash
sf probe 0 50000000; 

#从flash的0x100000地址读取0x4000长度的数据到内存0x80c00000
#即从flash的1M地址出读取16kb的内容到内存中，这部分内容为内核的dtb文件
sf read 0x80C00000 0x100000 0x4000; 

#从flash的0x110000地址读取0x400000长度的数据到内存0x80008000
#即从flash的1088K地址出读取4Mb的内容到内存中，这部分内容为内核的zImage文件
sf read 0x80008000 0x110000 0x400000;

#从内存的0x80008000启动内核
bootz 0x80008000 - 0x80C00000
```

> 所以
>
> zImage 应该写入 flash的1088K处
>
> suniv-f1c100s-licheepi-nano.dtb 应该写入flash的 1M处

```
sunxi-fel -p spiflash-write 0x100000 ./arch/arm/boot/dts/suniv-f1c100s-licheepi-nano.dtb
sunxi-fel -p spiflash-write 0x110000 ./arch/arm/boot/zImage
```

