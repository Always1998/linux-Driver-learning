# linux中断子系统

从stm32中断入手，到裸机的中断编写再到linux框架下的中断

## STM32中断

①、**中断向量表**：中断向量表存放的是**中断向量**。**中断服务程序的入口地址**或存放中断服务程序的首地址成为中断向量，因此中断向量表是一系列**中断服务程序入口地址**组成的表。这些中断服务程序(函数)在中断向量表中的位置是由半导体厂商定好的，当某个中断被触发以后就会自动跳转到中断向量表中对应的中断服务程序(函数)入口地址处。中断向量表在整个程序的最前面。

**中断向量表让CPU知道中断对应的中断服务函数。**

ARM 处理器都是从地址 0X00000000 开始运行 的，但是我们STM32 代码是下载到 0X8000000 开始的存储区域中。因此**中断向量表是存放到 0X8000000** 地址处的，Cortex-M 架构引入了一个新的概念——中断向量表偏移，通过**中断向量表偏移**就可以将中断向量表存放到**任意地址**处，中断向量表偏移配置通过向 **SCB_VTOR** 寄存器写入新的中断向量表首地址即可。

同样Cortex_A系列的内核也会有这两个概念，只是操作的寄存器不同罢了。

②、**NVIC(内嵌向量中断控制器)**。**使能和关闭指定的中断，设定中断的优先级。** 

Cortex-A系列的是**GIC中断控制器**。

③、**中断使能。**

④、**中断服务函数。**当中断发生以后中断服务函数就会被调用，我们要处理的工作就可以放到中断服务函数中去完成。

## linux裸机中断

 **中断向量表：** Cortex A7中断向量表，中断向量表也是在代码的最前面。

![image-20220404225241445](C:\Users\KY Xu\Desktop\picture\image-12)

可以看到CortexA系列的中断向量表只有八个，其中IRQ中断和复位中断最重要。Cortex-A 内核 CPU 的所有外部中断都属于这个 IRQ 中断，当任意一个外部中断发生的时候都会触发 IRQ 中断。在 IRQ 中断服务函数里面就可以读取指定的寄存器来判断发生的具体是什么中断，进而根据具体的中断做出相应的处理。

因此，以我们需要在 IRQ 中断服务函数中判断究竟是具体哪个外设中断发生了，然后做出相应的处理。

cortex M系列芯片的中断向量表公司已经写好了，cortex A系列裸机中断的向量表需要自己使用汇编语言去写

**中断控制器：**GIC V2 是给 ARMv7-A 架构使用的，比如 Cortex-A7、Cortex-A9、Cortex-A15 等，V3 和 V4 是给 ARMv8-A/R 架构使用的，也就是 64 位芯片使用。

GIC将中断分成3类：**SPI共享中断**，顾名思义，所有 Core 共享的中断，这个是最常见的，那些外部中断都属于 SPI 中断。比如按键中断、串口中断等等，这些中断所有的 Core 都可以处理，不限定特定 Core；中断ID为32至1019

​	**PPI私有中断**，我们说了 GIC 是支持多核的，每个核肯定有自己独有的中断。这些独有的中断肯定是要指定的核心处理，因此这些中断就叫做私有中断。中断ID为16-31

​	**SGI软件中断**，由软件触发引起的中断，通过向寄存器**GICD_SGIR** 写入数据来触发，系统会使用 SGI 中断来完成多核之间的通信。中断ID为0-15.

​	GIC 架构分为了两个逻辑块：Distributor 和 CPU Interface，也就是**分发器端**和 **CPU 接口端。**GIC 基地址以后只需要加上 **0X1000** 即可访问 GIC 分发器端寄存器,加上 **0X2000** 即可访问 GIC 的 CPU 接口段寄存器。GIC 控制器的寄存器基地址需要用到 Cortex-A 的 **CP15 协处理器**

​	CP15 协处理通过 **c0** 寄存器可以**获取到处理器内核信息**；通过 **c1 寄存器**可以**使能或禁止 MMU、I/D Cache** 等；通过 **c12** 寄存器可以设置**中断向量偏移**；通过 **c15** 寄存器可以**获取 GIC 基地址**



**中断使能**：中断使能包括两部分，一个是 I**RQ 或者 FIQ 总中断使能**，另一个就是 ID0~ID1019 这 **1020个中断源的使能**。总中断使能利用cpsid指令和cpsie指令来禁止和使能总中断，利用GIC 寄存器 **GICD_ISENABLERn** 和 **GICD_ ICENABLERn** 用来完成外部中断的使能和禁止



#### linux裸机中断向量表的实现

start.S,用于引导linux裸机中断。

``` c
.global _start
   
_start:
    ldr pc, =Reset_Handler /* 复位中断 */ 
    ldr pc, =Undefined_Handler /* 未定义指令中断 */
    ldr pc, =SVC_Handler /* SVC(Supervisor)中断*/
    ldr pc, =PrefAbort_Handler /* 预取终止中断 */
    ldr pc, =DataAbort_Handler /* 数据终止中断 */
    ldr pc, =NotUsed_Handler /* 未使用中断 */
	ldr pc, =IRQ_Handler /* IRQ 中断 */
	ldr pc, =FIQ_Handler /*FIQ中断*/
       
Reset_Handler:
    cpsid i /* 关闭全局中断 */

     /* 关闭 I,DCache 和 MMU 
     * 采取读-改-写的方式。
     */
    mrc p15, 0, r0, c1, c0, 0 /* 读取 CP15 的 C1 寄存器到 R0 中 */
    bic r0, r0, #(0x1 << 12) /* 清除 C1 的 I 位，关闭 I Cache */
    bic r0, r0, #(0x1 << 2) /* 清除 C1 的 C 位，关闭 D Cache */
    bic r0, r0, #0x2 /* 清除 C1 的 A 位，关闭对齐检查 */
    bic r0, r0, #(0x1 << 11) /* 清除 C1 的 Z 位，关闭分支预测 */
    bic r0, r0, #0x1 /* 清除 C1 的 M 位，关闭 MMU */
    mcr p15, 0, r0, c1, c0, 0 /*将 r0 的值写入到 CP15的C1中*/
        
#if 0
    /* 汇编版本设置中断向量表偏移 */
    ldr r0, =0X87800000
    dsb
    isb
    mcr p15, 0, r0, c12, c0, 0
    dsb
    isb
#endif
        
    /* 设置各个模式下的栈指针，
    * 注意：IMX6UL 的堆栈是向下增长的！
    * 堆栈指针地址一定要是 4 字节地址对齐的！！！
    * DDR 范围:0X80000000~0X9FFFFFFF 或者 0X8FFFFFFF
    */
    /* 进入 IRQ 模式 */
    mrs r0, cpsr
    bic r0, r0, #0x1f /* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
    orr r0, r0, #0x12 /* r0 或上 0x12,表示使用 IRQ 模式 */
    msr cpsr, r0 /* 将 r0 的数据写入到 cpsr 中 */
    ldr sp, =0x80600000 /* IRQ 模式栈首地址为 0X80600000,大小为 2MB */

    /* 进入 SYS 模式 */
    mrs r0, cpsr
    bic r0, r0, #0x1f /* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
    orr r0, r0, #0x1f /* r0 或上 0x1f,表示使用 SYS 模式 */
    msr cpsr, r0 /* 将 r0 的数据写入到 cpsr 中 */
    ldr sp, =0x80400000 /* SYS 模式栈首地址为 0X80400000,大小为 2MB */

    /* 进入 SVC 模式 */
    mrs r0, cpsr
    bic r0, r0, #0x1f /* 将 r0 的低 5 位清零，也就是 cpsr 的 M0~M4 */
    orr r0, r0, #0x13 /* r0 或上 0x13,表示使用 SVC 模式 */
    msr cpsr, r0 /* 将 r0 的数据写入到 cpsr 中 */
    ldr sp, =0X80200000 /* SVC 模式栈首地址为 0X80200000,大小为 2MB */

    cpsie i /* 打开全局中断 */

#if 0
	/* 使能 IRQ 中断 */
    mrs r0, cpsr /* 读取 cpsr 寄存器值到 r0 中 */
    bic r0, r0, #0x80 /* 将 r0 寄存器中 bit7 清零，也就是 CPSR 中的 I 位清零，表示允						许 IRQ 中断*/
    msr cpsr, r0 /* 将 r0 重新写入到 cpsr 中 */
#endif
     
    b main /* 跳转到 main 函数 */
 
/* 未定义中断 */
Undefined_Handler:
    ldr r0, =Undefined_Handler
    bx r0
 
/* SVC 中断 */
SVC_Handler:
    ldr r0, =SVC_Handler
    bx r0
 
/* 预取终止中断 */
PrefAbort_Handler:
    ldr r0, =PrefAbort_Handler 
    bx r0

/* 数据终止中断 */
DataAbort_Handler:
    ldr r0, =DataAbort_Handler
    bx r0

/* 未使用的中断 */
NotUsed_Handler:

    ldr r0, =NotUsed_Handler
    bx r0

/* IRQ 中断！重点！！！！！ */
IRQ_Handler:
	push {lr} /* 保存 lr 地址 */
	push {r0-r3, r12} /* 保存 r0-r3，r12 寄存器 */

	mrs r0, spsr /* 读取 spsr 寄存器 */
	push {r0} /* 保存 spsr 寄存器 */

	mrc p15, 4, r1, c15, c0, 0 /* 将 CP15 的 C0 内的值到 R1 寄存器中*/	
    add r1, r1, #0X2000 /* GIC 基地址加 0X2000，得到 CPU 接口端基地址 */
	ldr r0, [r1, #0XC] /* CPU 接口端基地址加 0X0C 就是 GICC_IAR 寄存器，
						* GICC_IAR 保存着当前发生中断的中断号，我们要根据
						*这个中断号来绝对调用哪个中断服务函数*/
	push {r0, r1} /* 保存 r0,r1 */ 
    
	cps #0x13 /* 进入 SVC 模式，允许其他中断再次进去 */

    push {lr} /* 保存 SVC 模式的 lr 寄存器 */
    ldr r2, =system_irqhandler /* 加载 C 语言中断处理函数到 r2 寄存器中*/
    blx r2 /* 运行 C 语言中断处理函数，带有一个参数 */

    pop {lr} /* 执行完 C 语言中断服务函数，lr 出栈 */
    cps #0x12 /* 进入 IRQ 模式 */
    pop {r0, r1} 
    str r0, [r1, #0X10] /* 中断执行完成，写 EOIR */

    pop {r0} 
    msr spsr_cxsf, r0 /* 恢复 spsr */

    pop {r0-r3, r12} /* r0-r3,r12 出栈 */
    pop {lr} /* lr 出栈 */
    subs pc, lr, #4 /* 将 lr-4 赋给 pc */

/* FIQ 中断 */
FIQ_Handler:

    ldr r0, =FIQ_Handler 
    bx r0    
 
```

1.使能中断，初始化相应的寄存器。

2.注册中断服务函数，也就是向 irqTable 数组的指定标号处写入中断服务函数

3.中断发生以后进入 IRQ 中断服务函数，在 IRQ 中断服务函数在数组 irqTable 里面查找具体的中断处理函数，找到以后执行相应的中断处理函数。



## linux中断

linux裸机实验中使用中断我们需要做大量工作，配置寄存器，使能 IRQ 等等。Linux 内核提供了完善的中断框架，我们只需要申请中断，然后注册中断处理函数即可，不需要一系列复杂的寄存器配置。

### api

``` C
//中断申请。request_irq函数可能会导致睡眠，因此不能在中断上下文或者其他禁止睡眠的代码段中使用 request_irq 函数。request_irq 函数会激活(使能)中断，所以不需要我们手动去使能中断
int request_irq(unsigned int irq, 
                irq_handler_t handler, 
                unsigned long flags,
                const char *name, 
                void *dev)
//中断释放。如果不是共享中断，那么该函数会删除中断处理函数并禁止中断
void free_irq(unsigned int irq, 
				void *dev);
//中断处理函数
irqreturn_t (*irq_handler_t) (int, void *);
/*第一个参数是要中断处理函数要相应的中断号。
 *第二个参数是一个指向 void 的指针，也就是个通用指针，需要与 request_irq 函数的 dev 参数	保持一致。用于区分共享中断的不同设备，dev 也可以指向设备数据结构。
  *中断处理函数的返回值为 irqreturn_t 类型*/

//中断使能与禁止
void enable_irq(unsigned int irq)
void disable_irq(unsigned int irq)
```

上面最简单的使能和禁止中断的API函数要等到当前正在执行的中断处理函数执行完才返回，因此使用者需要保证不会产生新的中断，并且确保所有已经开始执行的中断处理程序已经全部退出。在种情况下，可以使用另外一个中断禁止函数

``` c
void disable_irq_nosync(unsigned int irq)
```

该关闭中断函数调用以后立即返回，不会等待当前中断处理程序执行完毕。

上面的函数是关闭/使能某一个中断，如果是全局的中断的话，应该采用下面函数：

``` c
local_irq_enable();//local_irq_enable 用于使能当前处理器中断系统
local_irq_disable();//local_irq_disable 用于禁止当前处理器中断系统
```

### 中断上半部和下半部

使用**request_irq** 申请中断的时候注册的中断服务函数**属于中断处理的上半部**，只要中断触发，那么中断处理函数就会执行。断处理函数一定要快点执行完毕，越短越好，但是有些中断处理过程就是比较费时间。我们必须要对其进行处理，缩小中断处理函数的执行时间。

以电容屏触摸为例，电容触摸屏通过中断通知 SOC 有触摸事件发生，SOC 响应中断，然后通过 IIC 接口读取触摸坐标值并将其上报给系统。但是我们都知道 IIC 的速度最高也只有400Kbit/S，所以在中断中通过 IIC 读取数据就会浪费时间。我们可以将通过 IIC 读取触摸数据的操作暂后执行，中断处理函数仅仅相应中断，然后清除中断标志位即可。这个时候中断处理过程就分为了两部分：

**上半部**：上半部就是中断处理函数，那些处理过程比较快，不会占用很长时间的处理就可以放在上半部完成。

**下半部**：如果中断处理过程比较耗时，那么就将这些比较耗时的代码提出来，交给下半部去执行，这样中断处理函数就会快进快出。

Linux 内核将中断分为上半部和下半部的主要目的就是**实现中断处理函数的快进快出**，那些对时间敏感、执行速度快的操作可以放到中断处理函数中，也就是上半部。剩下的所有工作都可以放到下半部去执行。

1.如果要处理的内容**不希望被其他中断打断**，放到上半部

2.处理的任务对**时间敏感**，放到上半部

3.处理的任务和**硬件有关**，可以放到上半部（从内存中读取数据在上半部，对数据处理下半部）

#### 下半部机制

##### tasklet软中断

Linux 内核使用结构体 softirq_action 表示软中断。

``` c
struct softirq_action
{
 	void (*action)(struct softirq_action *);
};
```

softirq_action 结构体中的 action 成员变量就是软中断的服务函数。

对于linux内核而言，存在全局数组 softirq_vec 。他是记录软中断的数组，linux内核中有10种软中断机制，因此数组长度为10。

要使用软中断，必须先使用 open_softirq 函数注册对应的软中断处理函数，软中断必须在编译的时候静态注册。

``` c
void open_softirq(int nr, void (*action)(struct softirq_action *));

void raise_softirq(unsigned int nr);//触发软中断
```

TASKLET_SOFTIRQ**,** 即tasklet软中断是软中断的其中一种，推荐使用TASKLET软中断，在linux内核softirq_init函数找那个会默认打开tasklet,linux内核中使用tasklet_struct来表示tasklet软中断。

``` c
struct tasklet_struct
{
    struct tasklet_struct *next; /* 下一个 tasklet */
    unsigned long state; /* tasklet 状态 */
    atomic_t count; /* 计数器，记录对 tasklet 的引用数 */
    void (*func)(unsigned long); /* tasklet 执行的函数 */
    unsigned long data; /* 函数 func 的参数 */
};
```

要使用tasklet软中断，必须要先使用**tasklet_init**来初始化tasklet，api如下：

``` c
void tasklet_init(struct tasklet_struct *t,
                  void (*func)(unsigned long),
                  unsigned long data);
```

在上半部，也就是中断处理函数中调用 **tasklet_schedule** 函数就能使 tasklet 在合适的时间运行

``` c
void tasklet_schedule(struct tasklet_struct *t);
```

tasklet使用的模板

``` c
struct tasklet_struct tasklet_test;//定义tasklet

//tasklet处理函数
void tasklet_test_func(unsigned long data) {
    /*具体处理内容*/
}

//中断处理函数
irqreturn_t test_handler(int irq, void* dev_id) {
    /*中断具体的处理*/、
    tasklet_schedule(&tasklet_test);
    ........
}

static int __init xxx_init(void)//驱动入口函数
{
    tasklet_init(&tasklet_test, tasklet_test_func, data);
    request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
}
```



##### 工作队列

工作队列是另外一种下半部执行方式，工作队列在**进程上下文执行**，工作队列将要推后的工作交给一个**内核线程**去执行，因为工作队列工作在进程上下文，因此**工作队列允许睡眠或重新调度**。因此如果你**要推后的工作可以睡眠**那么就可以选择工作队列，否则的话就只能选择软中断或 tasklet。

Linux 内核使用 work_struct 结构体表示一个工作，

``` c
struct work_struct {
     atomic_long_t data; 
     struct list_head entry;
     work_func_t func; /* 工作队列处理函数 */
};
```

多个工作组织成一个工作队列，工作队列使用 workqueue_struct 结构体表示。

Linux 内核使用**工作者线程**(worker thread)来处理工作队列中的各个工作，Linux 内核使用worker 结构体表示工作者线程，worker结构体有一个需要关注的成员为rescue_wq,是一个工作队列

``` c
struct worker {
	...............
	struct workqueue_struct *rescue_wq;
};
```

每个 worker 都有一个**工作队列**，工作者线程处理自己工作队列中的所有工作。在实际的驱动开发中，我们只需要**定义工作**(work_struct)即可，关于工作队列和工作者线程我们基本不用去管，使用的api如下

``` c
//先创建工作，在进行初始化，初始化如下
#define INIT_WORK(_work, _func);

//工作的调度
bool schedule_work(struct work_struct *work);
```

工作队列使用模板

``` c
struct work_struct testwork;

void testwork_func_t(struct work_struct *work);
{
 /* work 具体处理内容 */
}
/* 中断处理函数 */
irqreturn_t test_handler(int irq, void *dev_id) {
 ......
 /* 调度 work */
 schedule_work(&testwork);
 ......
}
/* 驱动入口函数 */
static int __init xxxx_init(void) {
 ......
 /* 初始化 work */
 INIT_WORK(&testwork, testwork_func_t);
 /* 注册中断处理函数 */
 request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
 ......
}
```

##### 设备树中的中断属性

如果使用设备树的话就需要**在设备树中设置好中断属性信息**，Linux 内核通过读取设备树中的中断属性信息来配置中断。常用的中断属性有

①、#interrupt-cells，指定该中断控制器下描述中断信息的数量，对于GIC而言有3个cells。第一个 cells：**中断类型**，0 表示 SPI 中断，1 表示 PPI 中断。第二个 cells：**中断号**，对于 SPI 中断来说中断号的范围为 0~987，对于 PPI 中断来说中断号的范围为 0~15。第三个 cells：**标志**，bit[3:0]表示中断触发类型，为 1 的时候表示上升沿触发，为 2 的时候表示下降沿触发，为 4 的时候表示高电平触发，为 8 的时候表示低电平触发。bit[15:8]为 PPI中断的 CPU 掩码。

②、interrupt-controller，表示当前节点为中断控制器。

③、interrupts，指定中断号，触发方式等。按照父中断给定的interrupt-cells来指定

④、interrupt-parent，指定父中断。

如gpio中断控制器，在imx6ull.dtsi 中如下所示

``` c
gpio5: gpio@020ac000 { 
    compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio"; 
    reg = <0x020ac000 0x4000>;
    interrupts = <GIC_SPI 74 IRQ_TYPE_LEVEL_HIGH>,//GIC给定的interrupt-cells 3
    			<GIC_SPI 75 IRQ_TYPE_LEVEL_HIGH>;
    gpio-controller; //gpio子系统
    #gpio-cells = <2>;
    interrupt-controller; //gpio中断
    #interrupt-cells = <2>;
};
```

从设备树中获取设备号使用的API：

```c
//普适的从 interupts 属性中提取到对应的设备号
unsigned int irq_of_parse_and_map(struct device_node *dev,int index);

//GPIO获取相应的中断号
int gpio_to_irq(unsigned int gpio)
```

## 实验代码

``` c
VSCode中编写linux中断子系统
```







