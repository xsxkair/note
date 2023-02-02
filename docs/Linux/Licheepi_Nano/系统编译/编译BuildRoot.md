---
title: 编译BuildRoot
tags: teedoc, markdown, 语法
keywords: teedoc, markdown, 语法
desc: teedoc 的 markdown 语法介绍和实例
date: 2023-02-01 14:17:13
permalink: /pages/licheepi_nano/build_system/buildroot/
---

下载解压 buildroot

```
wget https://buildroot.org/downloads/buildroot-2021.02.4.tar.gz

tar xvf buildroot-2021.02.4.tar.gz

cd LicheePi_Nano/buildroot-2021.02.4
```

> 配置

```
make menuconfig

- Target options
  - Target Architecture (ARM (little endian))
  - Target Variant arm926t
  
- Toolchain
  - C library (glibc) # 使用glibc后，u-boot，linux，rootfs加起来的大小会超过flash16M，所以需要将rootfs放在TF卡中，但是glic比较通用，开发app比较方便，高手可以选其他
  
- System configuration
  - Use syslinks to /usr .... # 启用/bin, /sbin, /lib的链接
  - Enable root login # 启用root登录
  - Run a getty after boot # 启用登录密码输入窗口
  
  #因为荔枝派官方的内核版本为4.15，所以需要在buildroot中指定比这个低的才行
- buildroot
	- toolchain
		-kernel headers
			-4.14.x
```

> 烧录

```
因为flash的剩余空间不足以存放rootfs，所以将rootfs放在TF卡中，前面编译u-boot也已经设置为从tf卡加载文件系统
```

1. TF卡分区，只分一个区，从tf的开头开始

   ```
   gparted工具分区
   ```

2. 解压rootfs到tf卡

   ```
   tar -xvf buildroot-2021.02.4/output/images/rootfs.tar -C /media/xsx/rootfs/
   ```

   
