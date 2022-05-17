# linux I2C子系统驱动

I2C 是很常用的一个串行通信接口，用于连接各种外设、传感器等器件。

## 1.I2C的基础知识

i2c 支持一主多从，各设备地址独立，标准模式传输速率为 100kbit/s，快速模式为400kbit/s。I2C 物理总线使用两条总线线路，SCL：时钟线，数据收发同步。SDA：数据线，传输具体数据。

![image-20220516101948851](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220516101948851.png)

i2c的基本通信协议的格式如下所示：

4.总线时序

![image-20220401093947844](C:\Users\KY Xu\Desktop\picture\image-4)

​		数据起始位：当SCL为高电平的时候，SDA下降沿。表示S信号，传输开始

​		数据终止位：当SCL为高电平的时候，SDA上升沿。表示P信号，传输结束

​		数据传输：SCL为高电平的时候保证SDA数据稳定。数据传输时候MSB在前，LSB在后，从示波器上看，从左向右依次读出数据即可

​		应答信号：i2c 的数据字节定义为 8-bits 长度，对每次传送的总字节数量没有限制, 但对每一次传输必须伴有一个应答 (ACK) 信号，其时钟由主设备提供，而真正的应答信号由从设备发出，在**时钟为高时，通过拉低并保持 SDA 的值**来实现。如果从设备忙，它可以使 SCL 保持在低电平，这会强制使主设备进入等待状态。当从设备空闲后，并且释放时钟线，原来的数据传输才会继续。

​		 写时序：

![image-20220401094606000](C:\Users\KY Xu\Desktop\picture\image-6)

​		步骤：开始信号->发送IIC器件地址（7位地址+1位读写为）->从机ACK->重新发送开始信号->发送写入寄存器地址->从机ACK信号->发送写入寄存器的数据->从机ACK->停止信号

​        读时序：（分四步）

![image-20220401094939303](C:\Users\KY Xu\Desktop\picture\image-7)

​		步骤：开始信号->发送IIC器件地址+写标志位->从机ACK->重新发送开始信号->发送写入寄存器地址->从机ACK信号->重新发送开始位->发送IIC器件地址+读标志位->从机ACK->从机发送数据->主机ACK->停止信号	

## 2.I2C裸机驱动

对于i2c裸机驱动由两部分组成，分别是**IIC主机驱动**和**IIC设备驱动**。对于 I2C 主机驱动，对寄存器进行操作配置，一旦编写完成就不需要再做修改，其他的 I2C 设备直接调用主机驱动提供的 API 函数完成读写操作即可。

IIC主机驱动主机驱动配置**I2Cx_IADR**(x=1~4)寄存器，访问某个 I2C 从设备的时候就需要将其设备地址写入到 ADR 里面。配置**I2Cx_IFDR**寄存器，用来设置 I2C 的波特率。配置**I2Cx_I2CR**寄存器，设置I2C 使能、I2C 中断使能关闭、I2C 主从模式、传输方向、传输应答使能、重复开始信号。配置**I2Cx_I2SR、I2Cx_I2DR寄存器**。实现多个API接口如：初始化I2C，重新发送开始信号，发送开始信号，检查清楚错误，停止信号，发送数据，读取数据，IIC数据传输等接口。 

## 3.linux I2C驱动

Linux 的驱动分离与分层的思想，因此 Linux内核也将 I2C 驱动分为两部分：

①、I2C 总线驱动，I2C 总线驱动就是 SOC 的 I2C 控制器驱动，也叫做 I2C 适配器驱动。

②、I2C 设备驱动，I2C 设备驱动就是针对具体的 I2C 设备而编写的驱动

![image-20220516110105394](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220516110105394.png)

i2c 总线包括 i2c 设备 (i2c_client) 和 i2c 驱动 (i2c_driver), 当我们向 linux 中注册设备或驱动的时候，按照 i2c 总线匹配规则进行配对，配对成功，则可以通过 i2c_driver 中.prob 函数创建具体的设备驱动。在现代 linux 中，i2c 设备不再需要手动创建，而是使用设备树机制引入，设备树节点是与 paltform 总线相配合使用的。所以需先对 i2c 总线包装一层 paltform 总线，当设备树节点转换为平台总线设备时，我们在进一步将其转换为 i2c 设备，注册到 i2c 总线中。

设备驱动创建成功，我们还需要实现设备的文件操作接口,file_operations 中会使用到内核中 i2c 核心函数 (i2c 系统已经实现的**与具体硬件无关的函数**，专门开放给驱动工程师使用)。如i2c核心提供了i2c_adapter 和 i2c_driver的注册/注销函数

使用这些函数会涉及到 i2c 适配器，也就是 i2c 控制器。由于 ic2 控制器有不同的配置，所有 linux 将每一个 i2c控制器抽象成 i2c 适配器对象。这个对象中存在一个很重要的成员变量——Algorithm，Algorithm中存在一系列函数指针，这些函数指针指向真正硬件操作代码。

**I2C 总线**用到两个重要的数据结构：i2c_adapter 和 i2c_algorithm，如下所示

``` c
struct i2c_adapter {
	struct module *owner;
	unsigned int class;		  /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;

	/* data fields that are valid for all devices	*/
	const struct i2c_lock_operations *lock_ops;
	struct rt_mutex bus_lock;
	struct rt_mutex mux_lock;

	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */

	int nr;
	char name[48];
	struct completion dev_released;

	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;

	struct i2c_bus_recovery_info *bus_recovery_info;
	const struct i2c_adapter_quirks *quirks;

	struct irq_domain *host_notify_domain;
};
```

i2c_algorithm 类型的指针变量 algo是访问总线的算法。对于一个 I2C 适配器，肯定要对外提供读写 API 函数，设备驱动程序可以使用这些 API 函数来完成读写操作。i2c_algorithm 就是 I2C 适配器与 IIC 设备进行通信的方法。该结构体定义如下

``` c
struct i2c_algorithm {
	/* If an adapter algorithm can't do I2C-level access, set master_xfer
	   to NULL. If an adapter algorithm can do SMBus access, set
	   smbus_xfer. If set to NULL, the SMBus protocol is simulated
	   using common I2C messages */
	/* master_xfer should return the number of messages successfully
	   processed, or a negative value on error */
    //作为主设备时的发送函数
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num);
    //作为从设备时的发送函数。
	int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
			   unsigned short flags, char read_write,
			   u8 command, int size, union i2c_smbus_data *data);

	/* To determine what the adapter supports */
	u32 (*functionality) (struct i2c_adapter *);

#if IS_ENABLED(CONFIG_I2C_SLAVE)
	int (*reg_slave)(struct i2c_client *client);
	int (*unreg_slave)(struct i2c_client *client);
#endif
};
```

 I2C 适配器驱动的主要工作就是初始化 i2c_adapter 结构体变量，然后设置 i2c_algorithm 中的 master_xfer 函数。完成以后通过 **i2c_add_numbered_adapter**或 **i2c_add_adapter** 这两个函数向系统注册设置好的 i2c_adapter。

一般 SOC 的 I2C 总线驱动都是由半导体厂商编写的，不需要用户去编写。因此 I2C 总线驱动对我们这些 SOC 使用者来说是被屏蔽掉的，我们只要专注于 I2C 设备驱动即可。

**I2C设备驱动**重点关注两个数据结构：i2c_client 和 i2c_driver。

``` c
struct i2c_client {
	unsigned short flags;		/* div., see below		*/
	unsigned short addr;		/* chip address - NOTE: 7bit	*/
					/* addresses are stored in the	*/
					/* _LOWER_ 7 bits		*/
	char name[I2C_NAME_SIZE];
	struct i2c_adapter *adapter;	/* the adapter we sit on	*/
	struct device dev;		/* the device structure		*/
	int irq;			/* irq issued by device		*/
	struct list_head detected;
#if IS_ENABLED(CONFIG_I2C_SLAVE)
	i2c_slave_cb_t slave_cb;	/* callback for slave mode	*/
#endif
};
```

一个设备对应一个 i2c_client，每检测到一个 I2C 设备就会给这个 I2C 设备分配一个i2c_client。 

i2c_driver 类似 platform_driver，是编写 I2C 设备驱动重点要处理的内容

``` c
struct i2c_driver {
	unsigned int class;

	/* Standard driver model interfaces */
	int (*probe)(struct i2c_client *, const struct i2c_device_id *);
	int (*remove)(struct i2c_client *);

	/* New driver model interface to aid the seamless removal of the
	 * current probe()'s, more commonly unused than used second parameter.
	 */
	int (*probe_new)(struct i2c_client *);

	/* driver model interfaces that don't relate to enumeration  */
	void (*shutdown)(struct i2c_client *);

	/* Alert callback, for example for the SMBus alert protocol.
	 * The format and meaning of the data value depends on the protocol.
	 * For the SMBus alert protocol, there is a single bit of data passed
	 * as the alert response's low bit ("event flag").
	 * For the SMBus Host Notify protocol, the data corresponds to the
	 * 16-bit payload data reported by the slave device acting as master.
	 */
	void (*alert)(struct i2c_client *, enum i2c_alert_protocol protocol,
		      unsigned int data);

	/* a ioctl like command that can be used to perform specific functions
	 * with the device.
	 */
	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);
	//device_driver 驱动结构体，如果使用设备树的话，需要设置 device_driver 的
	//of_match_table 成员变量，也就是驱动的兼容(compatible)属性。
	struct device_driver driver;
    //传统不适用设备树时候的ID匹配表
	const struct i2c_device_id *id_table;

	/* Device detection callback for automatic device creation */
	int (*detect)(struct i2c_client *, struct i2c_board_info *);
	const unsigned short *address_list;
	struct list_head clients;

	bool disable_i2c_core_irq_mapping;
};
```

重点工作就是构建 i2c_driver，构建完成以后需要向Linux 内核注册这个 i2c_driver。

#### IIC的设备和驱动匹配

I2C 设备和驱动的匹配过程是由 I2C 核心来完成。 由**i2c_device_match** 函数实现

``` c
static int i2c_device_match(struct device *dev, struct device_driver *drv)
{
	struct i2c_client	*client = i2c_verify_client(dev);
	struct i2c_driver	*driver;


	/* Attempt an OF style match */
    //用于完成设备树设备和驱动匹配
	if (i2c_of_match_device(drv->of_match_table, client))
		return 1;

	/* Then ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	driver = to_i2c_driver(drv);

	/* Finally an I2C match */
    //传统的匹配方式。
	if (i2c_match_id(driver->id_table, client))
		return 1;

	return 0;
}
```

#### IIC的I2C 适配器驱动

I2C 适配器驱动就是 SOC 的 I2C 控制器驱动，一般由半导体厂商编写，I.MX6U 的 I2C 适配器驱动是个标准的 platform 驱动。当匹配完成后，会执行probe入口函数。

``` c
static int i2c_imx_probe(struct platform_device *pdev)
{
	const struct of_device_id *of_id = of_match_device(i2c_imx_dt_ids,
							   &pdev->dev);
	struct imx_i2c_struct *i2c_imx;
	struct resource *res;
	struct imxi2c_platform_data *pdata = dev_get_platdata(&pdev->dev);
	void __iomem *base;
	int irq, ret;
	dma_addr_t phy_addr;

	dev_dbg(&pdev->dev, "<%s>\n", __func__);
	//获取中断号
	irq = platform_get_irq(pdev, 0);
	if (irq < 0) {
		dev_err(&pdev->dev, "can't get irq number\n");
		return irq;
	}

    //数从设备树中获取 I2C1 控制器寄存器物理基地址，也就是 0X021A0000。
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    //获取到寄存器基地址以后使用 devm_ioremap_resource 函数对其进行内存映射，得到可以		在 Linux 内核中使用的虚拟地址。
	base = devm_ioremap_resource(&pdev->dev, res);
	if (IS_ERR(base))
		return PTR_ERR(base);

	phy_addr = (dma_addr_t)res->start;
    //申请内存。使用 imx_i2c_struct 结构体来表示 I.MX 系列 SOC 的 I2C 控制器
	i2c_imx = devm_kzalloc(&pdev->dev, sizeof(*i2c_imx), GFP_KERNEL);
	if (!i2c_imx)
		return -ENOMEM;

	if (of_id)
		i2c_imx->hwdata = of_id->data;
	else
		i2c_imx->hwdata = (struct imx_i2c_hwdata *)
				platform_get_device_id(pdev)->driver_data;

	/* Setup i2c_imx driver structure */
    //i2c_imx的adapter成员，就是适配器。关键在于配置适配器的algorithm成员。
	strlcpy(i2c_imx->adapter.name, pdev->name, sizeof(i2c_imx->adapter.name));
	i2c_imx->adapter.owner		= THIS_MODULE;
	i2c_imx->adapter.algo		= &i2c_imx_algo;
	i2c_imx->adapter.dev.parent	= &pdev->dev;
	i2c_imx->adapter.nr		= pdev->id;
	i2c_imx->adapter.dev.of_node	= pdev->dev.of_node;
	i2c_imx->base			= base;

	/* Get I2C clock */
	i2c_imx->clk = devm_clk_get(&pdev->dev, NULL);
	if (IS_ERR(i2c_imx->clk)) {
		dev_err(&pdev->dev, "can't get I2C clock\n");
		return PTR_ERR(i2c_imx->clk);
	}

	ret = clk_prepare_enable(i2c_imx->clk);
	if (ret) {
		dev_err(&pdev->dev, "can't enable I2C clock, ret=%d\n", ret);
		return ret;
	}

	/* Request IRQ */
    //注册控制器中断，i2c_imx_isr为中断处理程序。
	ret = devm_request_irq(&pdev->dev, irq, i2c_imx_isr,
				IRQF_SHARED | IRQF_NO_SUSPEND,
				pdev->name, i2c_imx);
	if (ret) {
		dev_err(&pdev->dev, "can't claim irq %d\n", irq);
		goto clk_disable;
	}

	/* Init queue */
	init_waitqueue_head(&i2c_imx->queue);

	/* Set up adapter data */
	i2c_set_adapdata(&i2c_imx->adapter, i2c_imx);

	/* Set up platform driver data */
	platform_set_drvdata(pdev, i2c_imx);

	pm_runtime_set_autosuspend_delay(&pdev->dev, I2C_PM_TIMEOUT);
	pm_runtime_use_autosuspend(&pdev->dev);
	pm_runtime_set_active(&pdev->dev);
	pm_runtime_enable(&pdev->dev);

	ret = pm_runtime_get_sync(&pdev->dev);
	if (ret < 0)
		goto rpm_disable;

	/* Set up clock divider */
    //IMX_I2C_BIT_RATE = 100kHz
	i2c_imx->bitrate = IMX_I2C_BIT_RATE;
	ret = of_property_read_u32(pdev->dev.of_node,
				   "clock-frequency", &i2c_imx->bitrate);
	if (ret < 0 && pdata && pdata->bitrate)
		i2c_imx->bitrate = pdata->bitrate;
	i2c_imx->clk_change_nb.notifier_call = i2c_imx_clk_notifier_call;
	clk_notifier_register(i2c_imx->clk, &i2c_imx->clk_change_nb);
	ret = i2c_imx_set_clk(i2c_imx, clk_get_rate(i2c_imx->clk));
	if (ret < 0) {
		dev_err(&pdev->dev, "can't get I2C clock\n");
		goto clk_notifier_unregister;
	}

	/*
	 * This limit caused by an i.MX7D hardware issue(e7805 in Errata).
	 * If there is no limit, when the bitrate set up to 400KHz, it will
	 * cause the SCK low level period less than 1.3us.
	 */
	if (is_imx7d_i2c(i2c_imx) && i2c_imx->bitrate > IMX_I2C_MAX_E_BIT_RATE)
		i2c_imx->bitrate = IMX_I2C_MAX_E_BIT_RATE;

	/* Set up chip registers to defaults */
    //配置寄存器i2cr和i2sr
	imx_i2c_write_reg(i2c_imx->hwdata->i2cr_ien_opcode ^ I2CR_IEN,
			i2c_imx, IMX_I2C_I2CR);
	imx_i2c_write_reg(i2c_imx->hwdata->i2sr_clr_opcode, i2c_imx, IMX_I2C_I2SR);

	/* Init optional bus recovery function */
	ret = i2c_imx_init_recovery_info(i2c_imx, pdev);
	/* Give it another chance if pinctrl used is not ready yet */
	if (ret == -EPROBE_DEFER)
		goto clk_notifier_unregister;

	/* Add I2C adapter */
    //向内核注册i2c_adapter。
	ret = i2c_add_numbered_adapter(&i2c_imx->adapter);
	if (ret < 0)
		goto clk_notifier_unregister;

	pm_runtime_mark_last_busy(&pdev->dev);
	pm_runtime_put_autosuspend(&pdev->dev);

	dev_dbg(&i2c_imx->adapter.dev, "claimed irq %d\n", irq);
	dev_dbg(&i2c_imx->adapter.dev, "device resources: %pR\n", res);
	dev_dbg(&i2c_imx->adapter.dev, "adapter name: \"%s\"\n",
		i2c_imx->adapter.name);
	dev_info(&i2c_imx->adapter.dev, "IMX I2C adapter registered\n");

	/* Init DMA config if supported */
    //申请DMA， I2C 适配器驱动采用了 DMA 方式。
	i2c_imx_dma_request(i2c_imx, phy_addr);

	return 0;   /* Return OK */

clk_notifier_unregister:
	clk_notifier_unregister(i2c_imx->clk, &i2c_imx->clk_change_nb);
rpm_disable:
	pm_runtime_put_noidle(&pdev->dev);
	pm_runtime_disable(&pdev->dev);
	pm_runtime_set_suspended(&pdev->dev);
	pm_runtime_dont_use_autosuspend(&pdev->dev);

clk_disable:
	clk_disable_unprepare(i2c_imx->clk);
	return ret;
}
```

其中，i2c_imx->adapter.algo = &i2c_imx_algo可以用来设置适配器中的通讯规则，他的实现如下所示：

``` c
static const struct i2c_algorithm i2c_imx_algo = {
	.master_xfer	= i2c_imx_xfer,
	.functionality	= i2c_imx_func,
};
```

其中functionality表征该总线适配的协议类型，master_xfer成员完成与 I2C 设备通信。

``` c
static int i2c_imx_xfer(struct i2c_adapter *adapter,
						struct i2c_msg *msgs, int num)
{
	unsigned int i, temp;
	int result;
	bool is_lastmsg = false;
	bool enable_runtime_pm = false;
	struct imx_i2c_struct *i2c_imx = i2c_get_adapdata(adapter);

	dev_dbg(&i2c_imx->adapter.dev, "<%s>\n", __func__);


	if (!pm_runtime_enabled(i2c_imx->adapter.dev.parent)) {
		pm_runtime_enable(i2c_imx->adapter.dev.parent);
		enable_runtime_pm = true;
	}

	result = pm_runtime_get_sync(i2c_imx->adapter.dev.parent);
	if (result < 0)
		goto out;

	/* Start I2C transfer */
    //开始IIC通信
	result = i2c_imx_start(i2c_imx);
	if (result) {
		if (i2c_imx->adapter.bus_recovery_info) {
			i2c_recover_bus(&i2c_imx->adapter);
			result = i2c_imx_start(i2c_imx);
		}
	}

	if (result)
		goto fail0;

	/* read/write data */
	for (i = 0; i < num; i++) {
		if (i == num - 1)
			is_lastmsg = true;

		if (i) {
			dev_dbg(&i2c_imx->adapter.dev,
				"<%s> repeated start\n", __func__);
			temp = imx_i2c_read_reg(i2c_imx, IMX_I2C_I2CR);
			temp |= I2CR_RSTA;
			imx_i2c_write_reg(temp, i2c_imx, IMX_I2C_I2CR);
			result = i2c_imx_bus_busy(i2c_imx, 1);
			if (result)
				goto fail0;
		}
		dev_dbg(&i2c_imx->adapter.dev,
			"<%s> transfer message: %d\n", __func__, i);
		/* write/read data */
..........................
    
    	//如果是从 I2C 设备读数据的话就调用 i2c_imx_read 函数
		if (msgs[i].flags & I2C_M_RD)
			result = i2c_imx_read(i2c_imx, &msgs[i], is_lastmsg);
        //向 I2C 设备写数据，如果要用 DMA 的话就使用 i2c_imx_dma_write 函数来完成写数据。如果不使用 DMA 的话就使用 i2c_imx_write 函数完成写数据
		else {
			if (i2c_imx->dma && msgs[i].len >= DMA_THRESHOLD)
				result = i2c_imx_dma_write(i2c_imx, &msgs[i]);
			else
				result = i2c_imx_write(i2c_imx, &msgs[i]);
		}
		if (result)
			goto fail0;
	}

fail0:
	/* Stop I2C transfer */
	i2c_imx_stop(i2c_imx);

	pm_runtime_mark_last_busy(i2c_imx->adapter.dev.parent);
	pm_runtime_put_autosuspend(i2c_imx->adapter.dev.parent);

out:
	if (enable_runtime_pm)
		pm_runtime_disable(i2c_imx->adapter.dev.parent);

	dev_dbg(&i2c_imx->adapter.dev, "<%s> exit with: %s: %d\n", __func__,
		(result < 0) ? "error" : "success msg",
			(result < 0) ? result : num);
	return (result < 0) ? result : num;
}
```

i2c_imx_start、i2c_imx_read、i2c_imx_write 和 i2c_imx_stop 这些函数就是对 I2C 寄存器的具体操作函数。



#### IIC设备驱动编写流程

这里主要指的是设备驱动的编写流程，针对使用设备树的内核场景。

**1.IIC设备信息描述**。创建相应的设备树节点。

**2.IIC设备数据收发处理**。在probe函数中调用i2c_transfer 函数。

``` c
int i2c_transfer(struct i2c_adapter *adap, 
					struct i2c_msg *msgs, 
						int num)
```

其中，i2c_msg是消息类型，在用 i2c_transfer 函数发送数据之前要先构建好 i2c_msg。i2c_msg具体结构如下：

``` c
struct i2c_msg {
	__u16 addr;	/* slave address			*/
	__u16 flags;
#define I2C_M_RD		0x0001	/* read data, from slave to master */
					/* I2C_M_RD is guaranteed to be 0x0001! */
#define I2C_M_TEN		0x0010	/* this is a ten bit chip address */
#define I2C_M_DMA_SAFE		0x0200/* the buffer of this message is DMA safe */
					/* makes only sense in kernelspace */
					/* userspace buffers are copied anyway */
#define I2C_M_RECV_LEN		0x0400	/* length will be first received byte */
#define I2C_M_NO_RD_ACK		0x0800	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_IGNORE_NAK	0x1000	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_REV_DIR_ADDR	0x2000	/* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_NOSTART		0x4000	/* if I2C_FUNC_NOSTART */
#define I2C_M_STOP		0x8000	/* if I2C_FUNC_PROTOCOL_MANGLING */
	__u16 len;		/* msg length				*/
	__u8 *buf;		/* pointer to msg data			*/
};
```

## 3.实验程序编写

1.修改设备树

2.驱动编写

在vscode中完成。

