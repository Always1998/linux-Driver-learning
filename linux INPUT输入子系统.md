# linux INPUT输入子系统

按键、鼠标、键盘、触摸屏等都属于输入(input)设备，Linux 内核为此专门做了一个叫做 input子系统的框架来处理输入事件。输入设备**本质上还是字符设备**，只是在此基础上**套上了 input 框架**。按键和键盘代表按键信息，鼠标和触摸屏代表坐标信息。input 子系统分为 **input 驱动**层、**input 核心层**、**input 事件处理层**，最终给用户空间提供可访问的设备节点

![image-20220515121939679](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220515121939679.png)

**驱动层**：输入设备的具体驱动程序，比如按键驱动程序，向内核层报告输入内容。

**核心层**：承上启下，为驱动层提供输入设备注册和操作接口。通知事件层对输入事件进行处理。

**事件层**：主要和用户空间进行交互。                                                       

## input子系统驱动编写流程

input 核心层会向 Linux 内核注册一个字符设备，input.c 就是 input 输入子系统的核心层。使用**class_register**函数来注册输入子系统设备的类，这样在启动后在sys/class目录下会有一个input目录。使用 **register_chrdev_region**函数来注册一个字符设备，这个该设备的设备号固定，是13。

步骤：

**1.注册input_dev**。我们在使用 input 子系统处理输入设备的时候就不需要去注册字符设备了，我们只需要向系统注册一个 input_device 即可。

``` c
struct input_dev {
	const char *name;
	const char *phys;
	const char *uniq;
	struct input_id id;

	unsigned long propbit[BITS_TO_LONGS(INPUT_PROP_CNT)];

	unsigned long evbit[BITS_TO_LONGS(EV_CNT)];//事件类型的位图
	unsigned long keybit[BITS_TO_LONGS(KEY_CNT)];//按键位图
	unsigned long relbit[BITS_TO_LONGS(REL_CNT)];//相对坐标位图
	unsigned long absbit[BITS_TO_LONGS(ABS_CNT)];//绝对坐标位图
	unsigned long mscbit[BITS_TO_LONGS(MSC_CNT)];//杂项事件位图
	unsigned long ledbit[BITS_TO_LONGS(LED_CNT)];//LED位图
	unsigned long sndbit[BITS_TO_LONGS(SND_CNT)];//sound有关的位图
	unsigned long ffbit[BITS_TO_LONGS(FF_CNT)];
	unsigned long swbit[BITS_TO_LONGS(SW_CNT)];
    .........................
	bool devres_managed;
};
```

其中，evbit 表示输入事件类型。共有18个事件，这18个事件分别为：

``` c
#define EV_SYN 0x00 /* 同步事件 */
#define EV_KEY 0x01 /* 按键事件！！！！ */
#define EV_REL 0x02 /* 相对坐标事件 */
#define EV_ABS 0x03 /* 绝对坐标事件 */
#define EV_MSC 0x04 /* 杂项(其他)事件 */
#define EV_SW 0x05 /* 开关事件 */
#define EV_LED 0x11 /* LED */
#define EV_SND 0x12 /* sound(声音) */
#define EV_REP 0x14 /* 重复事件 */
#define EV_FF 0x15 /* 压力事件 */
#define EV_PWR 0x16 /* 电源事件 */
#define EV_FF_STATUS 0x17 /* 压力状态事件 */
```

用到不同的事件时需要使用不同的位图。

在编写 input 设备驱动的时候我们需要先申请一个 input_dev 结构体变量，使用**input_allocate_device** 函数来申请一个 input_dev,要使用 **input_free_device** 函数来释放

```c
struct input_dev *input_allocate_device(void);
void input_free_device(struct input_dev *dev);
```

申请好一个 input_dev 以后就需要初始化这个 input_dev，需要初始化的内容主要为事件类型(evbit)和事件值(keybit)这两种。

input_dev 初始化完成以后就需要向 Linux 内核注册 input_dev了，需要用到 **input_register_device**。注销input_dev使用**input_unregister_device**

```c
int input_register_device(struct input_dev *dev);
void input_unregister_device(struct input_dev *dev);
```

**2.上报输入事件**。

当我们向 Linux 内核注册好 input_dev 以后还不能使用 input 设备，input 设备都是具有输入功能的，但是具体是什么样的输入值 Linux 内核是不知道的，我们需要获取到具体的输入值，**即输入事件**，然后将输入事件上报给 Linux 内核。**input_event** 函数可以上报所有的事件类型和事件值

```c
void input_event(struct input_dev *dev, 
                 unsigned int type, 
                 unsigned int code, 
                 int value)
```

由这个基本的函数封装产生了其他多个具体的事件上报函数。如下：

```c
void input_report_rel(struct input_dev *dev, unsigned int code, int value)
void input_report_abs(struct input_dev *dev, unsigned int code, int value)
void input_report_ff_status(struct input_dev *dev, unsigned int code, int value)
void input_report_switch(struct input_dev *dev, unsigned int code, int value)
void input_mt_sync(struct input_dev *dev)
    ........................
```

当我们上报事件以后还需要使用 **input_sync** 函数来告诉 Linux 内核 input 子系统上报结束，input_sync 函数本质是上报一个同步事件，此函数原型如下所示

#### input_event结构体

Linux 内核使用 input_event 这个结构体来表示所有的输入事件。input_envent 这个结构体非常重要，因为所有的输入设备最终都是按照 input_event 结构体呈现给用户的，用户应用程序可以通过 input_event 来获取到具体的输入事件或相关的值，比如按键值等。

``` c
struct input_event {
	struct timeval time;
	__u16 type;
	__u16 code;
	__s32 value;
};

//其中，timeval结构体如下.__kernel_time_t和__kernel_suseconds_t都是32位long型
struct timeval {
	__kernel_time_t		tv_sec;		/* seconds */
	__kernel_suseconds_t	tv_usec;	/* microseconds */
};
```

# input按键输入实验

见vscode实验。这里同中断实验，由于内核版本不同，其中的内核定时器的结构不同，在内核定时器这一块儿有着较大的差异。我的linux内核中timer_list如下：

``` c
struct timer_list {
	struct hlist_node	entry;
	unsigned long		expires;
	void			(*function)(struct timer_list *);
	u32			flags;

#ifdef CONFIG_LOCKDEP
	struct lockdep_map	lockdep_map;
#endif
};
```

所以在内核定时器的使用中有问题。

## linux中自带按键驱动程序

Linux 内核自带的 KEY 驱动文件为drivers/input/keyboard/gpio_keys.c，gpio_keys.c 采用了 platform 驱动框架，在 KEY 驱动上使用了 input 子系统实现。

``` c
static const struct of_device_id gpio_keys_of_match[] = {
	{ .compatible = "gpio-keys", },
	{ },
};
...................
static struct platform_driver gpio_keys_device_driver = {
	.probe		= gpio_keys_probe,
	.driver		= {
		.name	= "gpio-keys",
		.pm	= &gpio_keys_pm_ops,
		.of_match_table = gpio_keys_of_match,
	}
};

static int __init gpio_keys_init(void)
{
	return platform_driver_register(&gpio_keys_device_driver);
}

static void __exit gpio_keys_exit(void)
{
	platform_driver_unregister(&gpio_keys_device_driver);
}
```

有compatible属性可以知道，如果要使用设备树来描述 KEY 设备信息的话，设备节点的 compatible 属性值要设置为“gpio-keys”，当设备和驱动匹配以后 gpio_keys_probe 函数就会执行，该函数如下：

``` c
static int gpio_keys_probe(struct platform_device *pdev)
{
	struct device *dev = &pdev->dev;
	const struct gpio_keys_platform_data *pdata = dev_get_platdata(dev);
	struct fwnode_handle *child = NULL;
	struct gpio_keys_drvdata *ddata;
	struct input_dev *input;
	size_t size;
	int i, error;
	int wakeup = 0;

	if (!pdata) {
        //从设备树中获取到 KEY 相关的设备节点信息。
		pdata = gpio_keys_get_devtree_pdata(dev);
		if (IS_ERR(pdata))
			return PTR_ERR(pdata);
	}

	size = sizeof(struct gpio_keys_drvdata) +
			pdata->nbuttons * sizeof(struct gpio_button_data);
	ddata = devm_kzalloc(dev, size, GFP_KERNEL);
	if (!ddata) {
		dev_err(dev, "failed to allocate state\n");
		return -ENOMEM;
	}

	ddata->keymap = devm_kcalloc(dev,
				     pdata->nbuttons, sizeof(ddata->keymap[0]),
				     GFP_KERNEL);
	if (!ddata->keymap)
		return -ENOMEM;
	//使用 devm_input_allocate_device 函数申请 input_dev
	input = devm_input_allocate_device(dev);
	if (!input) {
		dev_err(dev, "failed to allocate input device\n");
		return -ENOMEM;
	}

	ddata->pdata = pdata;
	ddata->input = input;
	mutex_init(&ddata->disable_lock);

	platform_set_drvdata(pdev, ddata);
	input_set_drvdata(input, ddata);
	//初始化
	input->name = pdata->name ? : pdev->name;
	input->phys = "gpio-keys/input0";
	input->dev.parent = dev;
	input->open = gpio_keys_open;
	input->close = gpio_keys_close;

	input->id.bustype = BUS_HOST;
	input->id.vendor = 0x0001;
	input->id.product = 0x0001;
	input->id.version = 0x0100;

	input->keycode = ddata->keymap;
	input->keycodesize = sizeof(ddata->keymap[0]);
	input->keycodemax = pdata->nbuttons;

	/* Enable auto repeat feature of Linux input subsystem */
	if (pdata->rep)
        //设置 input_dev 事件，这里设置了 EV_REP 事件
		__set_bit(EV_REP, input->evbit);

	for (i = 0; i < pdata->nbuttons; i++) {
		const struct gpio_keys_button *button = &pdata->buttons[i];

		if (!dev_get_platdata(dev)) {
			child = device_get_next_child_node(dev, child);
			if (!child) {
				dev_err(dev,
					"missing child device node for entry %d\n",
					i);
				return -EINVAL;
			}
		}
		//继续设置key，设置input_dev的EV_KEY事件中KEY模拟为哪个按键
		error = gpio_keys_setup_key(pdev, input, ddata,
					    button, i, child);
		if (error) {
			fwnode_handle_put(child);
			return error;
		}

		if (button->wakeup)
			wakeup = 1;
	}

	fwnode_handle_put(child);

	error = devm_device_add_group(dev, &gpio_keys_attr_group);
	if (error) {
		dev_err(dev, "Unable to export keys/switches, error: %d\n",
			error);
		return error;
	}
	//向 Linux 系统注册 input_dev。
	error = input_register_device(input);
	if (error) {
		dev_err(dev, "Unable to register input device, error: %d\n",
			error);
		return error;
	}

	device_init_wakeup(dev, wakeup);

	return 0;
}
```

Linux 内核自带的 gpio_keys.c 驱动文件思路和我们前面编写的 keyinput.c 驱动文件基本一致。都是申请和初始化 input_dev，设置事件，向 Linux 内核注册 input_dev。最终在按键中断服务函数或者消抖定时器中断服务函数中上报事件和按键值。