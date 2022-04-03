# pinctrl和GPIO子系统

linux内核提供的pinctrl和gpio子系统来用于GPIO驱动开发，简化开发流程，避免对寄存器直接操作。Linux 驱动讲究驱动分离与分层，pinctrl 和 gpio 子系统就是驱动分离与分层思想下的产物，驱动分离与分层其实就是按照面向对象编程的设计思想而设计的设备驱动框架。

## pinctrl子系统

pinctrl子系统的作用：

①、获取设备树中 pin 信息。

②、根据获取到的 pin 信息来设置 pin 的复用功能

③、根据获取到的 pin 信息来设置 pin 的电气特性，比如上/下拉、速度、驱动能力等。

对于我们使用者来讲，只需要在设备树里面设置好某个 pin 的相关属性即可，其他的初始化工作均由 pinctrl 子系统来完成。

### iomuxc

``` c
//imx6ull.dtsi
iomuxc: iomuxc@020e0000 {
	compatible = "fsl,imx6ul-iomuxc";
	reg = <0x020e0000 0x4000>;
};
//imx6ull_emmc-npi.dts,更加具体
&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog_1>;

	pinctrl_hog_1: hoggrp-1 {
		fsl,pins = <
			MX6UL_PAD_UART1_RTS_B__GPIO1_IO19	0x17059 /* SD1 CD */
			MX6UL_PAD_GPIO1_IO05__USDHC1_VSELECT	0x17059 /* SD1 VSELECT */
			MX6UL_PAD_GPIO1_IO09__GPIO1_IO09        0x17059 /* SD1 RESET */
		>;
	};
..................
    pinctrl_led: ledgrp {
        fsl,pins = <
            MX6ULL_PAD_SNVS_TAMPER3__GPIO5_IO03 0x1b0b0
            >;
    };
};

```

不同的外设使用的 PIN 不同、其配置也不同。如果需要在 iomuxc 中添加我们自定义外设的 PIN，那么需要新建一个子节点，然后将这个自定义外设的所有 PIN 配置信息都放到这个子节点中。

其中宏定义的解释，比如MX6UL_PAD_UART1_RTS_B__GPIO1_IO19。这个通过在linux内核源码中搜索，可以得到

``` c
#define MX6UL_PAD_UART1_RTS_B__GPIO1_IO19		0x0090 0x031c 0x0000 5 0
```

这五个数值的元组代表<mux_reg  conf_reg  input_reg  mux_mode  input_val>。对于这五个数值的分析：

**0x0090**：mux_reg 寄存器偏移地址，设备树中的 iomuxc 节点就是 IOMUXC 外设对应的节点，根据其 reg 属性可知 IOMUXC 外设寄存器起始地址为 0x020e0000 。

**0x031c**：conf_reg 寄存器偏移地址。

**0x0000**：input_reg 寄存器偏移地址，有 input_reg 寄存器的外设需要配置 input_reg 寄存器。没有的话就不需要设置，UART1_RTS_B 这个 PIN 在做GPIO1_IO19 时没有 input_reg 寄存器。

**0x5** ： mux_reg 寄存器值

**0x0**：input_reg的值。

**0x17059**：他没有定义在宏中，是iomuxc节点中后面紧跟的值。代表config_reg的值。

所有的东西都已经准备好了，包括寄存器地址和寄存器值，Linux 内核相应的驱动文件就会根据这些值来做相应的初始化。linux内核驱动文件实现

![](C:\Users\KY Xu\Desktop\picture\pinctl框架图.drawio.png)

野火对于pin_ctrl子系统的驱动文件的结构的搭建。

### 在设备树中添加pinctrl节点模板

1.创建对应节点。注意！节点前缀一定要为“**pinctrl_**”

2.添加“fsl, pins”属性。属性名字一定要为**“fsl,pins”**，因为对于 I.MX 系列 SOC 而言，pinctrl 驱动程序是通过读取“fsl,pins”属性值来获取 PIN 的配置信息。

3.在"fsl, pins"属性中添加pin配置信息。一个五元组宏定义和config_reg寄存器的值



## gpio子系统

pinctrl 子系统重点是设置 PIN的复用和电气属性，如果 **pinctrl 子系统将一个 PIN 复用为 GPIO** 的话，那么接下来就要用到 gpio 子系统了。gpio 子系统顾名思义，就是用于初始化 GPIO 并且提供相应的 API 函数，比如设置 GPIO为输入输出，读取 GPIO 的值等。

**gpio 子系统的主要目的就是方便驱动开发者使用 gpio**，驱动开发者在设备树中添加 gpio 相关信息，然后就可以在驱动程序中使用 gpio 子系统提供的 API函数来操作 GPIO，Linux 内核向驱动开发者屏蔽掉了 GPIO 的设置过程，极大的方便了驱动开发者使用 GPIO。 

### 设备树中的gpio信息

在iomux子节点中，将引脚复用设置为GPIO。

``` c
fsl,pins = <
			MX6UL_PAD_UART1_RTS_B__GPIO1_IO19	0x17059 /* SD1 CD */
		>;
```

在usdhc1节点（SD卡设备总结点）中，并没有描述指定 CD 引脚的 pinctrl 信息。由于在“iomuxc”节点下引用了 pinctrl_hog_1 这个节点，所以 Linux 内核中的 iomuxc 驱动就会自动初始化 pinctrl_hog_1节点下的所有 PIN。

``` c
&usdhc1 {
	pinctrl-names = "default", "state_100mhz", "state_200mhz";
	pinctrl-0 = <&pinctrl_usdhc1>;
	pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
	pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
	cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;
	status = "okay";
};
```

根据cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>该句话，可以使用GPIO1_IO19。通过设备树gpio子节点，找到gpio驱动。

``` c
gpio1: gpio@209c000 {
    compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
    reg = <0x209c000 0x4000>;
    interrupts = <GIC_SPI 66 IRQ_TYPE_LEVEL_HIGH>,
    <GIC_SPI 67 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_GPIO1>;
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
    gpio-ranges = <&iomuxc  0 23 10>, <&iomuxc 10 17 6>,
    <&iomuxc 16 33 16>;
};
```

通过of_device_table匹配表查找compatible属性，找到驱动入口函数，完成gpio驱动

### GPIO API

对于驱动开发人员，设置好设备树以后就可以使用 gpio 子系统提供的 API 函数来操作指定的 GPIO，gpio 子系统向驱动开发人员屏蔽了具体的读写寄存器过程

``` c
//申请一个GPIO管脚，使用GPIO之前第一步
int gpio_request(unsigned gpio, const char *label);
//释放GPIO管脚，使用完之后释放
void gpio_free(unsigned gpio);
//此函数用于设置某个 GPIO 为输入
int gpio_direction_input(unsigned gpio);
//用于设置某个 GPIO 为输出，并且设置默认输出值
int gpio_direction_output(unsigned gpio, int value);
//数用于获取某个 GPIO 的值(0 或 1)
#define gpio_get_value __gpio_get_value
int __gpio_get_value(unsigned gpio);
//设置GPIO
#define gpio_set_value __gpio_set_value
void __gpio_set_value(unsigned gpio, int value)
```

### GPIO设备树模板

1.创建设备节点。在根节点下创立子节点

2.添加pinctrl信息。

``` c
test {
  	pinctrl_name = "default";
    pinctrl-0 = <&pinctrl_test>;
    /*其他*/
};
```

3.添加GPIO属性信息

``` c
test {
  	pinctrl_name = "default";
    pinctrl-0 = <&pinctrl_test>;
    gpio = <&gpio1 0 GPIO_ACTIVE_HIGH>;
};
```

## gpio led驱动编写

``` C
VScode中编写
```

同样有相同步骤的蜂鸣器驱动编写，按键输入驱动这些只和GPIO有关的驱动

作为输入的GPIO只需要实现file_operation中的open和read即可，不需要实现write。
