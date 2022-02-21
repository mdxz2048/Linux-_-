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



### IOMUX寄存器

## 编写LED驱动框架
