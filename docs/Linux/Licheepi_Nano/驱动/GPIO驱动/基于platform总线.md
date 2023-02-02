---
title: 基于Platform驱动模型
date: 2023-02-01 14:17:13
permalink: /pages/licheepi_nano/driver/led/platform/
---
## 编写设备树

```
platform_led {
		compatible = "xsx,platform_led_dri";
		#address-cells = <1>;
		#size-cells = <1>;
		reg = <0X01C20800 0x04 0X01C20810 0x04>;
		status = "okay";
	};
```

## 编写驱动

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
#include <linux/of_platform.h>
#include <linux/of_address.h>

static dev_t led_dev_num;         // 定义一个设备号
static struct cdev *led_cdev;     // 定义一个设备管理结构体指针
static struct class *led_class;   // 定义一个设备类
static struct device *led_device; // 定义一个设备

volatile unsigned long *gpio_a_cfg0;
volatile unsigned long *gpio_a_data;

static int led_open(struct inode *inode, struct file *file)
{
    *((volatile size_t *)gpio_a_cfg0) &= ~(7 << 0); // 清除配置寄存器
    *((volatile size_t *)gpio_a_cfg0) |= (1 << 0);  // 配置GPIOE10为输出模式

    return 0;
}

static int led_close(struct inode *inode, struct file *file)
{
    return 0;
}

static int led_read(struct file *filp, char __user *buff, size_t count, loff_t *off)
{
    int res;
    size_t status = *((volatile size_t *)gpio_a_data);

    // 将内核空间数据拷贝到用户空间
    // statue => buff, 大小4字节
    res = copy_to_user(buff, &status, 4);
    if (res < 0)
        printk(KERN_DEBUG "read error!!!\r\n");
    else
        printk(KERN_DEBUG "read ok!!!\r\n");

    return res;
}

static int led_write(struct file *filp, const char __user *buff, size_t count, loff_t *offp)
{
    int res = 0;
    size_t statue = 0;

    res = copy_from_user(&statue, buff, 4);
    if (res < 0)
    {
        printk(KERN_DEBUG "write error!!!\r\n");
        return 0;
    }

    if (!!statue)
        *((volatile size_t *)gpio_a_data) |= (1 << 0);
    else
        *((volatile size_t *)gpio_a_data) &= ~(1 << 0);

    return 1;
}

static struct file_operations led_ops = {
    .owner = THIS_MODULE,
    .open = led_open,
    .read = led_read,
    .write = led_write,
    .release = led_close,
};

static int led_probe(struct platform_device * pdev)
{
    struct resource * res;
    int ret;

    led_cdev = cdev_alloc();    //动态申请一个设备结构体
    if(led_cdev == NULL) {
        printk(KERN_WARNING "cdev alloc fail!!\r\b");
        return -1;
    }

    ret = alloc_chrdev_region(&led_dev_num, 0, 1, "led_num");//动态申请一个设备号
    if(ret != 0) {
        printk(KERN_WARNING "alloc_chrdev_region fail!!\r\b");
        return -2;
    }

    led_cdev->owner = THIS_MODULE;
    led_cdev->ops = &led_ops;
    cdev_add(led_cdev, led_dev_num, 1);    //将设备添加到内核中

    led_class = class_create(THIS_MODULE, "led_class");
    if(led_class == NULL) {
        printk(KERN_WARNING "class_create fail!!\r\b");
        return -3;
    }

//这个名字会作为驱动设备的文件名
    led_device = device_create(led_class, NULL, led_dev_num, NULL, "led_device");
    if(IS_ERR(led_device)) {
        printk(KERN_WARNING "device_create fail!!\r\b");
        return -4;
    }

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    gpio_a_cfg0 = ioremap(res->start, res->end - res->start + 1);

    res = platform_get_resource(pdev, IORESOURCE_MEM, 1);
    gpio_a_data = ioremap(res->start, res->end - res->start + 1);

    return 0;
}

static int led_remove(struct platform_device * pdev)
{
    iounmap(gpio_a_cfg0);
    iounmap(gpio_a_data);

    cdev_del(led_cdev);
    unregister_chrdev_region(led_dev_num, 1);
    device_destroy(led_class, led_dev_num);
    class_destroy(led_class);

    return 0;
}

//优先级高：匹配设备树中compatible属性全部
static struct of_device_id led_match_table[]={
    {
        .compatible = "xsx,platform_led_dri",
    },
    {},
};
//优先级低于of_match_table：匹配设备树中compatible属性中的设备名
static struct platform_device_id led_device_ids[]={
    {
        .name="platform_led_dri",
    },
    {},
};

static struct platform_driver led_driver={
    .probe = led_probe,
    .remove = led_remove,
    .driver = {
        .name = "led",
        .of_match_table = led_match_table,
    },
    .id_table = led_device_ids,
};

static int led_driver_init(void)
{
    platform_driver_register(&led_driver);

    return 0;
}

static void led_driver_exit(void){
    platform_driver_unregister(&led_driver);
}

module_init(led_driver_init);
module_exit(led_driver_exit);

MODULE_LICENSE("GPL");          //不加的话加载会有错误提醒
MODULE_AUTHOR("1477153217@qq.com");     //作者
MODULE_VERSION("0.1");          //版本
MODULE_DESCRIPTION("led_driver");  //简单的描述

```

```
#在makefile中加入
obj-y		+= xsx_platform_led.o

#然后编译烧录系统即可在/dev下发现led_device文件
```



## 测试程序

```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main(int argc, char **argv)
{
    int fd;
    char *filename = NULL;
    int val, cnt, i;

    filename = argv[1];
    fd = open(filename, O_RDWR);
    if (fd < 0)
    {
        printf("err, can`t open %s\r\n", filename);
        return 0;
    }

    if (argc != 3)
    {
        printf("usage: ./led_app.exe [device] [次数]\r\n");
    }

    cnt = strtol(argv[2], NULL, 10);
    for (i = 0; i < cnt; i++)
    {
        val = 0;
        write(fd, &val, 4);
        sleep(1);

        val = 1;
        write(fd, &val, 4);
        sleep(1);
    }

    close(fd);
    return 0;
}
```

```
#编译app
arm-linux-gcc led_test.c -o led_test_app.o

#下载到板子上
chmod 777 ./led_test_app.o
./led_test_app.o /dev/led_device 10
LED即可闪烁10次
```

