# linux LCD驱动子系统

LCD 全称是 Liquid Crystal Display，液晶显示器是常用到的外设，通过 LCD 可以显示绚丽的图形、界面等，提高人机交互的效率，在裸机驱动中I.MX6U 提供了一个 **eLCDIF** 接口用于连接 RGB 接口的液晶屏

## 1.裸机驱动中的LCD驱动介绍

#### LCD中的几个概念

1.**分辨率**。 

LCD 显示器分辨率。LCD 显示器都是由一个一个的像素点组成。720p：1280×720个像素点；1080P：1920×1080个像素点；2k：2560×1440；4k：3840×2160。

2.**像素格式**

RGB888：R、 G、B 这三部分分别使用 8bit 的数据。ARGB8888：加入 8bit 的 Alpha(透明)通道

3.**LCD屏幕接口**

I.MX6U-ALPHA支持**RGB接口**的LCD，RGB接口和VGA没什么区别。**VGA信号**的组成分为五种：红绿蓝三原色和行场同步信号。R/G/B：八位。VSYNC/HSYNC：垂直/水平同步信号。DE：数据使能。PCLK：像素时钟线。

4.**LCD时间参数**

以720p分辨率的屏幕为例，显示的过程中就是用一根“笔”在不同的像素点画上不同的颜色。这根笔按照从左至右、从上到下的顺序扫描每个像素点，并且在像素画上对应的颜色。

![image-20220515195149903](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220515195149903.png)

**HSYNC** 产生，表示开始显示新的一行；**VSYNC** 就表示开始显示新的一帧图像。

**HBP、HFP、VBP 和 VFP** 就是导致图中黑边的原因。由于 RGB LCD屏幕内部是有一个 IC 的，发送一行或者一帧数据给 IC，IC 是需要反应时间的。通过这段反应时间可以让 IC 识别到一行数据扫描完了，要换行了；或者一帧图像扫描完了，要开始下一帧图像显示。

5.**RGB LCD屏幕时序**

显示一行的数据时序如下所示：

![image-20220515195740441](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220515195740441.png)

HSYNC拉低表示一行的开始，HSPW是 HSYNC 信号宽度。HBP是行同步信号后肩，HFP为行同步信号前肩。HSYNC 信号发出以后，需要等待 **HSPW+HBP** 个 CLK 时间才会接收到真正有效的像素数据。当显示完一行数据以后需要等待 HFP 个 CLK 时间才能发出下一个 HSYNC 信号，所以**显示一行所需要的时间就是：HSPW + HBP + HOZVAL + HFP。**

显示一帧的时序如下所示：

![image-20220515201129121](C:\Users\KY Xu\AppData\Roaming\Typora\typora-user-images\image-20220515201129121.png)

VSYNC低电平有效，表示一帧的开始，VSPW是VSYNC的宽度。**VBP**是帧同步信号后肩，单位为 1 行的时间。**LINE：**是显示一帧有效数据所需的时间，假如屏幕分辨率为 1024*600，那么 LINE 就是 600 行的时间。**VFP**是帧同步信号前肩。显示一帧所需要的时间就是：VSPW+VBP+LINE+VFP 个行时间，最终的计算公式：

​							**T = (VSPW+VBP+LINE+VFP) * (HSPW + HBP + HOZVAL + HFP)**

对于不同分辨率的屏幕，时间参数都不同，屏幕制造商会给出各时间参数。

6.**显存**

ARGB8888 格式的话一个像素需要 4 个字节的内存来存放像素数据，那么 1024×600 分辨率就需要 1024×600×4=2457600B≈2.4MB 内存。但是 RGB LCD 内部是没有内存的，所以就需要在开发板上的 DDR3 中分出一段内存作为 RGB LCD 屏幕的显存，在屏幕上显示图像直接操作这部分显存即可。

#### eLCDIF接口

eLCDIF 是 I.MX6U 自带的液晶屏幕接口，用于连接 RGB LCD 接口的屏幕。eLCDIF 支持三种接口：**MPU 接口**、**VSYNC 接口**和 **DOTCLK 接口**。

**MPU接口**：MPU 接口用于在 I.MX6U 和 LCD 屏幕直接传输数据和命令。

**VSYNC接口**：VSYNC 接口时序和 MPU 接口时序基本一样，只是多了 VSYNC 信号来作为帧同步。

**DOTCLK接口**：DOTCLK 接口就是用来连接 RGB LCD 接口屏幕的， 它包括 VSYNC、HSYNC、DOTCLK和 ENABLE(可选的)这四个信号。DOTCLK接口时序同上面VGA接口时序。

eLCDIF 要驱动起来 RGB LCD 屏幕，重点是**配置好时间参数**，这个通**过配置相应的寄存器**实现。关键寄存器： **LCDIF_CTRL**，**LCDIF_CTRL1**，**LCDIF_TRANSFER_COUNT**， 

**LCDIF_VDCTRL0~4**寄存器组。 

#### 裸机LCD驱动编写步骤

①、初始化 I.MX6U 的 eLCDIF 控制器，重点是 LCD 屏幕宽(width)、高(height)、hspw、hbp、hfp、vspw、vbp 和 vfp 等信息。

②、初始化 LCD 像素时钟。

③、设置 RGBLCD 显存。

④、应用程序直接通过操作显存来操作 LCD，实现在 LCD 上显示字符、图片等信息。

## 2. linux LCD驱动

在 Linux 中应用程序最终也是通过操作 RGB LCD 的显存来实现在 LCD 上显示字符、图片等信息。在裸机中我们在DDR中分配显存，但是在 Linux 系统中内存的管理很严格，显存是需要申请的。而且因为虚拟内存的存在，**驱动程序设置的显存**和**应用程序访问的显存**要是**同一片物理内存**。

为了解决上述问题，Framebuffer 诞生了， Framebuffer 翻译过来就是帧缓冲，简称 fb。fb 是一种机制，将系统中所有跟显示有关的硬件以及软件集合起来，虚拟出一个 fb 设备，当我们编写好 LCD 驱动以后会生成一个名为/dev/fbx(x=0~n)的设备，应用程序通过访问/dev/fbx 这个设备就可以访问 LCD。fb设备是一个字符设备。

#### LCD驱动解析

不同分辨率的 LCD 屏幕其 eLCDIF 控制器驱动代码都是一样的，只需要修改好对应的屏幕参数即可。屏幕参数信息属于屏幕设备信息内容，这些肯定是要放到设备树中的，因此主要工作就是**修改设备树**，NXP 官方的设备树已经添加了 LCD 设备节点，只是此节点的 LCD 屏幕信息是针对 NXP 官方 EVK 开发板所使用的 4.3 寸 480*272 编写的，因此需要按需修改。NXP提供的设备树节点如下：

``` c
lcdif: lcdif@21c8000 {
    compatible = "fsl,imx6ul-lcdif", "fsl,imx28-lcdif";
    reg = <0x21c8000 0x4000>;
    interrupts = <GIC_SPI 5 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_LCDIF_PIX>,
    <&clks IMX6UL_CLK_LCDIF_APB>,
    <&clks IMX6UL_CLK_DUMMY>;
    clock-names = "pix", "axi", "disp_axi";
    status = "disabled";
};
```

上面lcdif 节点信息是所有使用 I.MX6ULL 芯片的板子所**共有**的，并**不是完整的 lcdif 节点信息**。像屏幕参数这些需要根据不同的硬件平台去添加。

LCD驱动是一个标准的的 platform 驱动，当驱动和设备匹配以后mxsfb_probe 函数就会执行。Linux 内核将所有的 Framebuffer 抽象为一个叫做 fb_info 的结构体，fb_info 结构体包含了 Framebuffer 设备的完整属性和操作集合，因此每一个 Framebuffer 设备都必须有一个 fb_info。换言之就是，LCD 的驱动就是构建 fb_info，并且向系统注册 fb_info的过程。

linux LCD驱动中的mxsfb_probe()函数工作流程如下：

①、申请 fb_info。

②、初始化 fb_info 结构体中的各个成员变量。

③、初始化 eLCDIF 控制器。

④、使用 **register_framebuffer** 函数向 Linux 内核注册初始化好的 fb_info。

``` c
int register_framebuffer(struct fb_info *fb_info)
```

简单分析mxsfb_probe(）函数如下：

``` c
static int mxsfb_probe(struct platform_device *pdev)
{
	const struct of_device_id *of_id =
			of_match_device(mxsfb_dt_ids, &pdev->dev);
	struct resource *res;
    //host 结构体指针变量，表示 I.MX6ULL 的 LCD 的主控接口。这是一个设备结构体，包含了I.MX系列SOC的FrameBuffer设备的详细信息
	struct mxsfb_info *host;
	struct fb_info *fb_info;
	struct pinctrl *pinctrl;
	int irq = platform_get_irq(pdev, 0);
	int gpio, ret;
.....................
    //从设备树中获取 eLCDIF 接口控制器的寄存器首地址，设备树中 lcdif 节点已经设置了 eLCDIF 寄存器首地址为 0X021C8000，因此 res=0X021C8000
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (!res) {
		dev_err(&pdev->dev, "Cannot get memory IO resource\n");
		return -ENODEV;
	}

    //给 host 申请内存，host 为 mxsfb_info 类型结构体指针
	host = devm_kzalloc(&pdev->dev, sizeof(struct mxsfb_info), GFP_KERNEL);
	if (!host) {
		dev_err(&pdev->dev, "Failed to allocate IO resource\n");
		return -ENOMEM;
	}

    //给fb_info申请内存
	fb_info = framebuffer_alloc(0, &pdev->dev);
	if (!fb_info) {
		dev_err(&pdev->dev, "Failed to allocate fbdev\n");
		devm_kfree(&pdev->dev, host);
		return -ENOMEM;
	}
    //设置 host 的 fb_info 成员变量为 fb_info，设置 fb_info 的 par 成员变量为
    //host。通过这一步就将前面申请的 host 和 fb_info 联系在了一起。
	host->fb_info = fb_info;
	fb_info->par = host;

    //申请中断。
	ret = devm_request_irq(&pdev->dev, irq, mxsfb_irq_handler, 0,
			  dev_name(&pdev->dev), host);
	if (ret) {
		dev_err(&pdev->dev, "request_irq (%d) failed with error %d\n",
				irq, ret);
		ret = -ENODEV;
		goto fb_release;
	}

    //对从设备树中获取到的寄存器首地址(res)进行内存映射，得到虚拟地址，并保存到 host 的 base 成员变量。通过base和偏移值实现访问颞部的eLCDIF寄存器。
	host->base = devm_ioremap_resource(&pdev->dev, res);
	if (IS_ERR(host->base)) {
		dev_err(&pdev->dev, "ioremap failed\n");
		ret = PTR_ERR(host->base);
		goto fb_release;
	}
.................................
    //给pseudo_palette申请内存，pseudo_palette是调色板
	fb_info->pseudo_palette = devm_kcalloc(&pdev->dev, 16, sizeof(u32),
					       GFP_KERNEL);
	if (!fb_info->pseudo_palette) {
		ret = -ENOMEM;
		dev_err(&pdev->dev, "Failed to allocate pseudo_palette memory\n");
		goto fb_release;
	}

	INIT_LIST_HEAD(&fb_info->modelist);

	pm_runtime_enable(&host->pdev->dev);

    //初始化fb_info
	ret = mxsfb_init_fbinfo(host);
	if (ret != 0) {
		dev_err(&pdev->dev, "Failed to initialize fbinfo: %d\n", ret);
		goto fb_pm_runtime_disable;
	}

	ret = mxsfb_dispdrv_init(pdev, fb_info);
	if (ret != 0) {
		if (ret == -EPROBE_DEFER)
			dev_info(&pdev->dev,
				 "Defer fb probe due to dispdrv not ready\n");
		goto fb_free_videomem;
	}

	if (!host->dispdrv) {
		pinctrl = devm_pinctrl_get_select_default(&pdev->dev);
		if (IS_ERR(pinctrl)) {
			ret = PTR_ERR(pinctrl);
			goto fb_pm_runtime_disable;
		}
	}

	if (!host->enabled) {
        //设置eLCDIF寄存器
		writel(0, host->base + LCDC_CTRL);
		mxsfb_set_par(fb_info);
		mxsfb_enable_controller(fb_info);
		pm_runtime_get_sync(&host->pdev->dev);
	}

    //向linux内核注册fb_info。
	ret = register_framebuffer(fb_info);
	if (ret != 0) {
		dev_err(&pdev->dev, "Failed to register framebuffer\n");
		goto fb_destroy;
	}
.....................................
	return ret;
}
```

**mxsfb_init_fbinfo** 函数通过调用 mxsfb_init_fbinfo_dt 函数**从设备树中获取到 LCD 的各个参数信息**。最后，**mxsfb_init_fbinfo**函数会调用 mxsfb_map_videomem 函数申**请 LCD 的帧缓冲内存(也就是显存)。** 

#### LCD驱动使用

LCD的驱动NXP已经写好了，我们需要做的就是按照所使用的 LCD 来修改设备树。重点要注意三个地方：

①、LCD 所使用的 **IO 配置**。

②、**LCD 屏幕节点修改**，修改相应的属性值，换成我们所使用的 LCD 屏幕参数。

③、**LCD 背光节点信息修改**，要根据实际所使用的背光 IO 来修改相应的设备节点信息。