# platform 设备驱动

前面介绍的驱动编写都是简单的读写接口的驱动，但是后续的SPI，IIC等复杂的外设不能采用这么去写了。linux驱动要考虑驱动的可重用性，因此提出了**驱动设备分离**的思想。因此诞生了平台设备驱动**platform设备驱动**。

## 1.linux驱动的分离和分层

假如现在有三个平台 A、B 和 C，这三个SoC上都有 MPU6050 这 个 I2C 接口的六轴传感器，如果每个每个平台都有一个MPU6050的驱动。

![image-20220513092757839](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220513092757839.png)

每个平台上都有一个主机驱动和一个设备驱动。**主机驱动是必要的**，不同的平台其 IIC 控制器不同。但是右侧的设备驱动就没必要每个平台都写一个，因为不管对于那个 SOC 来说，MPU6050 都是一样，通过 I2C 接口读写数据就行了，**只需要一个 MPU6050 的设备驱动程序**即可。

因此可以法就是每个平台的 I2C 控制器都提供一个统一的接口(主机驱动)，每个设备的话也只提供一个驱动程序(设备驱动)，每个设备通过统一的 I2C接口驱动来访问。

![image-20220513093229813](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220513093229813.png)



这个就是**驱动的分离**，也就是将主机驱动和设备驱动分隔开来。在实际的驱动开发中，一般 IIC 主机控制器驱动已经由半导体厂家编写好了，而设备驱动一般也由设备器件的厂家编写好了，我们只需要提供设备信息。

将设备信息从设备驱动中剥离开来，驱动使用标准方法去获取到设备信息(比如从设备树中获取到设备信息)，然后根据获取到的设备信息来初始化设备。 这样就相当于**驱动只负责驱动，设备只负责设备，想办法将两者进行匹配即可**。这个就是 Linux 中的总线(bus)、驱动(driver)和设备(device)模型，也就是常说的驱动分离。**总线就是驱动和设备信息的月老，负责给两者牵线搭桥**。

![image-20220513093643863](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220513093643863.png)

当注册驱动的时候在右侧设备中查找，看是否有匹配设备。通过相应的匹配规则进行匹配。

## 2.platform平台驱动模型

在 SOC 中有些外设是没有总线这个概念的，为了解决此问题，Linux 提出了 **platform 这个虚拟总线**，相应的就有 platform_driver 和 platform_device。linux中用bus_type表示总线，platform 总线是 bus_type 的一个具体实例。

``` c
struct bus_type platform_bus_type = { 
	name = "platform", 
    dev_groups = platform_dev_groups, 
    match = platform_match, 
    uevent = platform_uevent, 
    pm = &platform_dev_pm_ops, 
};
```

match 函数设备和驱动之间匹配的，总线就是使用 match 函数来**根据注册的设备来查找对应的驱**

**动，或者根据注册的驱动来查找相应的设备**

match 函数有两个参数：**dev 和 drv**，这两个参数分别为 device 和 device_driver 类型。

match函数有四种匹配方式：**设备树采用的匹配方式**：device_driver 结构体(表示设备驱动)中有个名为of_match_table的成员变量，此成员变量保存着驱动的**compatible匹配表**，设备树中的每个设备节点的 compatible 属性会和 of_match_table 表中的所有成员比较，查看是否有相同的条目，如果有的话就表示设备和此驱动匹配，设备和驱动匹配成功以后 probe 函数就会执行。

**ACPI匹配**。**id_table匹配**：每个 platform_driver 结构体有一个 id_table成员变量，保存了很多 id 信息。通过id信息判断是否匹配。

**name匹配**： 匹配platform_driver 结构体和platform_device结构体的name字段是否相等。

### 2.1 platform 驱动

``` c
struct platform_driver {
    //probe 函数，当驱动与设备匹配成功以后 probe 函数就会执行,一般驱动的提供者会编写=
    int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
    //device_driver 结构体变量.device_driver 相当于基类，提供了最基础的驱动框架     		//plaform_driver 继承了这个基类，然后在此基础上又添加了一些特有的成员变量。
	struct device_driver driver;
	//id_table 是个表(也就是数组)，每个元素的类型为 platform_device_id，
    const struct platform_device_id *id_table; 		 
    bool prevent_deferred_probe;
};
```

上面提到了device_driver 基类，device_driver定义中有一些关键点：

``` c
struct device_driver {
    const char* name;
    struct bus_type* bus;
    .....
    //是采用设备树的时候驱动使用的匹配表，同样是数组，每个匹配项都为 of_device_id 结构体
    const struct of_device_id *of_match_table;
    .....
}

struct of_device_id { 
    char name[32];
	char type[32];
    //匹配的属性值。过设备节点的 compatible 属性值和 of_match_table 中每个项目的 compatible 成员变量进行比较
	char compatible[128];
 	const void *data; 
};
```

platform_driver驱动程序编写的步骤如下：

1.定义一个 platform_driver 结构体变量，然后实现结构体中的各个成员变量，重点是实现匹配方法以及 probe 函数。

2.要在驱动入口函数里面调用platform_driver_register 函数向 Linux 内核注册一个 platform 驱动

``` c
int platform_driver_register (struct platform_driver *driver)
```

3.在驱动卸载函数中通过 platform_driver_unregister 函数卸载 platform 驱动。

``` c
void platform_driver_unregister(struct platform_driver *drv)
```

驱动框架如下所示

``` c
struct xxx_dev {
    struct cdev cdev;
    /*设备结构体其他的内容*/
};

struct xxx_dev xxxdev;

static int xxx_open(struct inode *inode, struct file *flip) {
    /*open接口具体内容*/
    return 0;
}

static ssize_t xxx_write(struct inode *inode, struct file *flip) {
    /*write接口具体内容*/
    return 0;
}

static struct file_operations xxx_fops = {
    .owner = THIS_MODULE,
    .open = xxx_open,
    .write = xxx_write,
};

//platform驱动的probe函数
static int xxx_probe(struct platform_device* dev) {
    cdev_init(&xxxdev, &xxx_fops);//注册字符设备驱动
    /*函数功能实现*/
    return 0;
}
//，当关闭 platform备驱动的时候此函数就会执行，以前在驱动卸载 exit 函数里面要做的事情就放到此函数中来
static int xxx_remove(struct platform_device*dev) {
    cdev_del(&xxxdev.cdev);
    /*函数功能实现*/
    return 0;
}

//匹配列表
static const struct of_device_id xxx_of_match[] = {
    {.compatible = "xxx-gpio"},
    {/*Sential*/}
};

//platform平台驱动结构体
static struct platform_driver xxx_driver = {
    .driver ={
        .name = "xxx",
        .of_match_table = xxx_of_match,
    },
    .probe = xxx_probe,
    .remove = xxx_remove,
};

static int __init xxxdriver_init(void) {
    return platform_driver_register(&xxx_driver);
}

static void __exit xxxdriver_exit(void) {
    platform_driver_unregister(&xxx_driver);
}

module_init(xxxdriver_init);
module_exit(xxxdriver_exit);

MODULE_LICENCE("GPL");
MODULE_AUTHOR("xky");
```

### 2.2 platform 设备

platform_device 这个结构体表示 platform 设备，这里我们要注意，如果内核支持设备树的话就不要再使用 platform_device 来描述设备了，因为改用设备树去描述了。因此不做太多介绍，都会去用设备树。

## platform 设备驱动程序实验

这里主要关注设备树下的驱动编写。在vscode中实现。

同样在linux内核中实现了led-gpio的驱动，这里面给出关键代码的源码分析。

``` c
static int gpio_led_probe(struct platform_device *pdev)
{
	struct gpio_led_platform_data *pdata = dev_get_platdata(&pdev->dev);
	struct gpio_leds_priv *priv;
	int i, ret = 0;
	//非设备数方式可以省略不看
	if (pdata && pdata->num_leds) {
		priv = devm_kzalloc(&pdev->dev,
				sizeof_gpio_leds_priv(pdata->num_leds),
					GFP_KERNEL);
		if (!priv)
			return -ENOMEM;

		priv->num_leds = pdata->num_leds;
		for (i = 0; i < priv->num_leds; i++) {
			ret = create_gpio_led(&pdata->leds[i], &priv->leds[i],
					      &pdev->dev, NULL,
					      pdata->gpio_blink_set);
			if (ret < 0)
				return ret;
		}
	}
    //设备树方式，重点关注！！！
    else {
		priv = gpio_leds_create(pdev);
		if (IS_ERR(priv))
			return PTR_ERR(priv);
	}

	platform_set_drvdata(pdev, priv);

	return 0;
}

static void gpio_led_shutdown(struct platform_device *pdev)
{
	struct gpio_leds_priv *priv = platform_get_drvdata(pdev);
	int i;

	for (i = 0; i < priv->num_leds; i++) {
		struct gpio_led_data *led = &priv->leds[i];

		if (!(led->cdev.flags & LED_RETAIN_AT_SHUTDOWN))
			gpio_led_set(&led->cdev, LED_OFF);
	}
}

static struct platform_driver gpio_led_driver = {
	.probe		= gpio_led_probe,
	.shutdown	= gpio_led_shutdown,
	.driver		= {
		.name	= "leds-gpio",
		.of_match_table = of_gpio_leds_match,
	},
};

module_platform_driver(gpio_led_driver);
```

1.module_platform_driver函数，完成platform驱动的注册和删除。

2.gpio_leds_create函数从设备树中提取设备信息。其函数定义为

``` c
static struct gpio_leds_priv *gpio_leds_create(struct platform_device *pdev)
```

