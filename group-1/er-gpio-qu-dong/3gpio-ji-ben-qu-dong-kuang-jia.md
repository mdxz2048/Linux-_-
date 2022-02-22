# 3-GPIO基本驱动框架

本节我们主要分为三个部分，梳理硬件操作逻辑、查找手册确定寄存器地址、编写LED驱动框架；

## 梳理硬件操作逻辑

首先，我们需要控制LED2的亮和熄灭，要明确的是需要使用的c引脚的GPIO功能，并且需要配置为输出模式。参考本章第2节中对于GPIO编程的介绍，我们的硬件操作逻辑可以梳理为：

1. 确保时钟源ipg_clk_s已经被打开，可以提供稳定时钟；
2. 配置IOMUX寄存器，将引脚GPIO5_3设置为GPIO；
3. 配置GPIOx_GDIR寄存器，将引脚GPIO5_3设置为输出；
4. 通过GPIO_DR寄存器控制GPIO5_3输出的高、低电平，来实现控制LED2的亮、熄灭。

## GPIO5_3查找手册确定寄存器地址

### 时钟源-ipg_clk_s

1. 我们继续查阅imux6ull的数据手册的`28.3章节`可以看到，GPIO时钟源如图；

    ![image-20220221212442032](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220221212442032.png)

2. 这里，我们主要关注CCM模块的ipg_clk_root是否被打开，通过手册中下表，可以看到CCM模块的`CCGR1[CG15]`被用来使能GPIO5的时钟；

    

    ![image-20220221214005621](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220221214005621.png)

3. CCM的CCGR全称是Clock Gating Register，即时钟门控寄存器，寄存器每2位来作为一组使用，代表是否向一个模块提供时钟，具体2位的含义如下：

​	

| CGR数值 | 含义                                                         |
| ------- | ------------------------------------------------------------ |
| 00      | 该模块时钟关闭，硬件握手被禁用                               |
| 01      | CPU运行在run mode模式时，该模块时钟使能；CPU运行在 WAIT和STOP模式时，该模块时钟被禁止； |
| 10      | 保留                                                         |
| 11      | 除了CPU运行在STOP模式，其他该模块时钟均被使能；              |

4. 通过imux6ull手册中关于CCM_CCGR1的介绍，我们可以看到该寄存器地址为**20C_406Ch**；

![](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220222193810079.png)

5. 我们这里需要将**CCM_CCGR1**寄存器(20C_406Ch)的CG15(30、31位)设置为11即可打开GPIO5的时钟；

### IOMUXC寄存器

1. 对于某个/某组功能，IOMUX有2个寄存器来配置；

    - IOMUXC_SW_MUX_CTL_PAD_<PADNAME>	选择某个pad功能
    - IOMUXC_SW_MUX_CTL_GRP_<GROUP NAME>  选择某组功能

    某个引脚，或是某组预设的引脚，都有8个可选的模式(alternate (ALT) MUX_MODE)。

    ![image-20220222201830654](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220222201830654.png)

2. 设置上下拉电阻等参数：

    - IOMUXC_SW_PAD_CTL_PAD_<PAD_NAME>：pad pad xxx，设置某个pad的参数_
    - IOMUXC_SW_PAD_CTL_GRP_<GROUP NAME>：pad grp xxx，设置某组引脚的参数

3. 其他

### GPIO内部寄存器

GPIO寄存器的详细描述见《28.5 GPIO Memory Map/Register Definition》的表，其中GPIO5的相关寄存器截图如下：

![image-20220222203127380](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220222203127380.png)

1. GPIOx_GDIR寄存器地址为20AC004 + 4 = **20AC008h**，这里我们的PAD对应的是GPIO5_3,因此我们需要将第3位2设置位1，输出模式；

    ![image-20220222203441993](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220222203441993.png)

    

2. GPIO_DR寄存器用来控制输出是高还是低，地址是20AC000+0 = **20AC000h**，我们在输出高电平的时候，给第3位写1即可，输出低电平的时候，给第3位写0即可；

    ![image-20220222203631392](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220222203631392.png)

3. QITA



## 编写LED驱动框架
