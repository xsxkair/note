---
title: 基于GPIO子系统的LED驱动
tags: LED驱动
keywords: LED驱动
desc: LED驱动
date: 2023-02-01 14:17:13
permalink: /pages/licheepi_zero/driver/led/gpio/
---

## 1. 驱动系统自带LED驱动
    ##全部disabled
    leds {
		blue_led {
			status = "disabled";
		};

		green_led {
			status = "disabled";
		};

		red_led {
			status = "disabled";
		};
	};

## 2. 添加自定义led节点
    bingpi_led_in_gpio {
		compatible = "xsx,bingpi_gpio";
		led-gpios = <&pio 6 1 GPIO_ACTIVE_LOW>; /* PG1 */
		status = "okay";
	};

## 3. 添加驱动
> driver/leds/led_bingpi_in_gpio.c
```
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/io.h>
#include <linux/uaccess.h>
#include <linux/device.h>
#include <linux/cdev.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/gpio/consumer.h>
#include <linux/slab.h>
#include <linux/string.h>
#include <linux/gpio.h>

static dev_t led_dev_num; // 定义一个设备号
static struct cdev *led_cdev; // 定义一个设备管理结构体指针
static struct class *led_class; // 定义一个设备类
static struct device *led_device; // 定义一个设备

struct gpio_desc *led_pin;

static int led_open(struct inode *inode, struct file *file)
{
	gpiod_direction_output(led_pin, 0);
	return 0;
}

static int led_close(struct inode *inode, struct file *file)
{
	return 0;
}

static int led_read(struct file *filp, char __user *buff, size_t count,
		    loff_t *off)
{
	int res;
	size_t status = gpiod_get_value(led_pin);

	// 将内核空间数据拷贝到用户空间
	// statue => buff, 大小4字节
	res = copy_to_user(buff, &status, 4);
	if (res < 0)
		printk(KERN_DEBUG "read error!!!\r\n");
	else
		printk(KERN_DEBUG "read ok!!!\r\n");

	return res;
}

static int led_write(struct file *filp, const char __user *buff, size_t count,
		     loff_t *offp)
{
	int res = 0;
	size_t statue = 0;

	res = copy_from_user(&statue, buff, 4);
	if (res < 0) {
		printk(KERN_DEBUG "write error!!!\r\n");
		return 0;
	}

	if (!!statue)
		gpiod_set_value(led_pin, 1);
	else
		gpiod_set_value(led_pin, 0);

	return 1;
}

static struct file_operations led_ops = {
	.owner = THIS_MODULE,
	.open = led_open,
	.read = led_read,
	.write = led_write,
	.release = led_close,
};

static int led_probe(struct platform_device *pdev)
{
	struct resource *res;
	int ret;

	led_cdev = cdev_alloc(); //动态申请一个设备结构体
	if (led_cdev == NULL) {
		printk(KERN_WARNING "cdev alloc fail!!\r\b");
		return -1;
	}

	ret = alloc_chrdev_region(&led_dev_num, 0, 1,
				  "led_bingpi_gpio"); //动态申请一个设备号
	if (ret != 0) {
		printk(KERN_WARNING "alloc_chrdev_region fail!!\r\b");
		return -2;
	}

	led_cdev->owner = THIS_MODULE;
	led_cdev->ops = &led_ops;
	cdev_add(led_cdev, led_dev_num, 1); //将设备添加到内核中

	led_class = class_create(THIS_MODULE, "led_bingpi_gpio_class");
	if (led_class == NULL) {
		printk(KERN_WARNING "class_create fail!!\r\b");
		return -3;
	}

	led_device = device_create(led_class, NULL, led_dev_num, NULL,
				   "led_bingpi_gpio");
	if (IS_ERR(led_device)) {
		printk(KERN_WARNING "device_create fail!!\r\b");
		return -4;
	}

	led_pin = devm_gpiod_get(&pdev->dev, "led", GPIOF_OUT_INIT_LOW);
	if (IS_ERR(led_pin)) {
		printk(KERN_ERR "Get gpio resource failed!\n");
		return -1;
	}

	return 0;
}

static int led_remove(struct platform_device *pdev)
{
	devm_gpiod_put(&pdev->dev, led_pin);

	return 0;
}

static struct of_device_id led_match_table[] = {
	{
		.compatible = "xsx,bingpi_gpio",
	},
	{},
};

static struct platform_device_id led_device_ids[] = {
	{
		.name = "bingpi_gpio",
	},
	{},
};

static struct platform_driver led_driver={
    .probe = led_probe,
    .remove = led_remove,
    .driver = {
        .name = "led_gpio",
        .of_match_table = led_match_table,
    },
    .id_table = led_device_ids,
};

module_platform_driver(led_driver);

MODULE_LICENSE("GPL"); //不加的话加载会有错误提醒
MODULE_AUTHOR("1477153217@qq.com"); //作者
MODULE_VERSION("0.1"); //版本
MODULE_DESCRIPTION("led_driver"); //简单的描述

```

## 4. 修改makefile
> obj-y		+= led_bingpi_in_gpio.o

## 5. 编译后下载验证
设备路径：/dev/led_bingpi_gpio，存在即表明驱动注册成功
app验证：`./led_test_app.o /dev/led_bingpi_gpio 10`
现象：LED闪10次