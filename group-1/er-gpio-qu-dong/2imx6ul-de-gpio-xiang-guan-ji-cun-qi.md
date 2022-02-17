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

### GPIO编程

#### 读模式

1. 配置IOMUX寄存器选择引脚位GPIO模式(通过IOMUX可以控制(IOMUXC)寄存器)；

2. 配置GPIO方向寄存器((GPIO_GDIR[GDIR])对应位设置为0，配置方向为input；

3. 从数据寄存器(GPIO_DR)或者PAD状态寄存器(GPIO_PSR)读取输入信号；

   注：

   - 当GPIO_GDIR = 0,即方向设置为input时，读取GPIO_DR寄存器返回的是GPIO_PSR寄存器对应位的值，即输入信号。

#### 写模式

1. 配置IOMUX寄存器选择引脚位GPIO模式(通过IOMUX可以控制(IOMUXC)寄存器)。如果需要通过PSR寄存器确认写入的值，则需要使能SION；
2. 配置GPIO_GDIR寄存器对应位为1b，设置为输出模式；
3. 写数据到GPIO_DR寄存器；

#### 中断控制单元

中断检测单元是通过读取GDIR寄存器的值，来检测在配置的GPIO外设对应引脚上，支持上升沿触犯、下降沿触发、高电平触发和低电平触发。

GPIO_ICR1和GPIO_ICR2作为中断控制寄存器来对GPIO的中断控制功能进行配置。

### GPIO寄存器

#### 内存地址定义

参考数据手册《28.5 GPIO Memory Map/Register Definition》内容。

![image.png](https://ae03.alicdn.com/kf/H50218755d1294ffdb54dbe7d0aba119eY.png)

![image.png](https://ae02.alicdn.com/kf/H4742809c1b8c48fd8ae19b6c942980e4Q.png)

#### 寄存器描述

##### GPIO data register (GPIOx_DR)

- GPIO_DR数据寄存器是用来临时保存数据。如果IOMUXC寄存器处于GPIO模式，且GPIO的方向被设置，GPIO_DR寄存器的数据会输出到对应位。

- 如果GPIO的方向位被清空，再读取GPIO_DR寄存器将反应对应位的信号；

- 两个读取动作之间需要等待两个周期。

  ![image.png](https://ae05.alicdn.com/kf/H7d15da13e76b4163ac793dc58cda66eeN.png)

GPIO_DR位的读取与IOMUXC 输入模式和GDIR位的设置如下：

- GDIR[n]被设置，IOMUXC处于GPIO模式，读取 DR[n]返回当前的 DR[n]；
- GDIR[n] 被清零，IOMUXC处于GPIO模式，读取 DR[n]返回对应位的输入信号的值；
- GDIR[n]被设置，IOMUXC没有处于GPIO模式，读取 DR[n]返回当前的 DR[n]；
- GDIR[n] 被清零，IOMUXC没有处于GPIO模式，读取DR[n]总是返回0；

##### GPIO direction register (GPIOx_GDIR)

- 当IOMUXC被设置位GPIO模式时，GPIO_GDIR功能是方向控制器；
- GPIO_GDIR每个位为对应引脚指定一个方向；
- 每个控制信号对应SoC上的对应引脚，由SoC和IOMUX对应位来决定实际的方向(是否有效)；

![image.png](https://ae02.alicdn.com/kf/H998f832cb7b9490fb15afaef34f4dbcax.png)

## 开发板原理图
