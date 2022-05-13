# linux MISC驱动开发

MISC驱动，即杂项驱动。当板子上的某些外设无法进行分类的时候就可以使用 MISC 驱动。MISC 驱动其实就是最简单的字符设备驱动，通常嵌套在 platform 总线驱动中，实现复杂的驱动。

## 1.MISC驱动简介

所有MISC设备驱动的主设备号都是**10**，不同的设备他们的从设备号是不同的。MISC设备会自动创建cdev，不用我们手动创建。因此采用MISC驱动可以简化字符设备驱动的编写。

MISC驱动编写时，需要向linux中注册一个miscdevice结构体。

``` c
struct miscdevice  {
	int minor;
	const char *name;
	const struct file_operations *fops;
	struct list_head list;
	struct device *parent;
	struct device *this_device;
	const struct attribute_group **groups;
	const char *nodename;
	umode_t mode;
};
```

定义一个 MISC 设备(miscdevice 类型)以后我们需要设置 minor、name 和 fops 这三个成员变量。minor 表示子设备号，需要用户指定子设备号,Linux 系统已经预定义了一些 MISC 设备的子设备号(0-255)。name 就是此 MISC 设备名字，当此设备注册成功以后就会在/dev 目录下生成一个名为 name的设备文件。fops 就是字符设备的操作集合，MISC 设备驱动最终是需要使用用户提供的 fops操作集合。
设置好miscdevice后需要使用misc_register来向系统注册一个MISC设备:

``` c
int misc_register(struct miscdevice * misc)
```

用这一个函数即可实现之前所有的字符设备注册的五步

``` c
1 alloc_chrdev_region(); /* 申请设备号 */
2 cdev_init(); /* 初始化 cdev */
3 cdev_add(); /* 添加 cdev */
4 class_create(); /* 创建类 */
5 device_create(); /* 创建设备 */
```

同理，用misc_deregister()函数就可以代替字符设备后续销毁处理的四步。

## 2.驱动程序实验

在vscode中实现，

