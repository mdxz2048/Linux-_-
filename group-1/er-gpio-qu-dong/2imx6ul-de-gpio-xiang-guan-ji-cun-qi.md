# 2-imx6ul的GPIO相关寄存器

## 数据手册

首先需要下载SOC的数据手册，我们这里使用的是**IMX6ULLRM.pdf**。

### IOMUX方案

![](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/7ddbd3518a93df7c8dc3fc5d963a37e.png)

如上图所示，PAD1/PAD2可以理解为我们引脚，从左至右(从上至下)为IOMUX的分解，主要包括了IOMUX的三部分：

- MUX_MODE模块，控制IOMUX的模式，这里主要控制引脚是否使用为GPIO功能(因为引脚可能还会被复用在Block的其他功能，比如作为串口的某个引脚)；
- pad settings模块，设置引脚,这里不使用该模块设置引脚；
- GPIO模块寄存器部分，这里主要包括8个寄存器：Data register数据寄存器 (GPIO_DR)、 GPIO direction register方向寄存器 (GPIO_GDIR)、 Pad sample引脚采样寄存器 register (GPIO_PSR)、Interrupt control registers中断控制寄存器 (GPIO_ICR1, GPIO_ICR2)、Edge select register边沿选择寄存器 (GPIO_EDGE_SEL)、Interrupt mask register中断屏蔽寄存器 (GPIO_IMR)和Interrupt status register终端状态寄存器 (GPIO_ISR)。

### 功能

#### GPIO的输入/输出功能

- 通过配置数据寄存器(GPIO_DR)将数据写入到输出；
- 通过配置方向寄存器(GPIO_GDIR)控制GPIO的方向；
- 通过读取引脚采样寄存器(GPIO_PSR)的对应位来读取输入信号；

#### GPIO中断

- 最多支持 32 个中断；
- 识别中断边缘；
- 生成3个高电平中断向SOC的中断控制器(Generates three active-high interrupts to the SoC interrupt controller)

### 外部信号描述

| Instance | Signal   |
| -------- | -------- |
| GPIO1    | IO0~IO31 |
| GPIO2    | IO0~IO21 |
| GPIO3    | IO0~IO28 |
| GPIO4    | IO0~IO28 |
| GPIO5    | IO0~IO11 |

### 时钟源

GPIO的时钟源如下。

| Clock name | Clock Root   | Description             |
| ---------- | ------------ | ----------------------- |
| ipg_clk_s  | ipg_clk_root | Peripheral access clock |



## 开发板原理图
