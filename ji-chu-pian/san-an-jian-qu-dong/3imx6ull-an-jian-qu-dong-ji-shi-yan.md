# 3-imx6ull按键驱动及实验

我们这里分为4部分完成本节

- 实验演示;
- 按键原理图，通过查看原理图来确定具体要控制的引脚;
- GPIO控制器，在GPIO章节已经介绍过GPIO控制器，我们结合相关知识，来确定我们在按键驱动程序中需要操作、读取的寄存器；
- 驱动程序，这里按照驱动的加载顺序、应用层接口调用顺序来进行梳理；

## 实验演示

## 按键原理图

1. 开发板的原理图按键部分如下，确定板子上按键对应的GPIO引脚,可以看到`KEY1`对应`GPIO5_1`、`KEY2`对应`GPIO4_14`引脚；

    ![image-20220409112228376](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409112228376.png)

2. 同时分析原理图电路，可以得到如下关系:

    | 按键状态 | 引脚状态 | 数据寄存器 |
    | -------- | -------- | ---------- |
    | 按下     | 低电平   | 0          |
    | 抬起     | 高电平   | 1          |

    

3. 至此,按键原理图部分就先到这里；

## 寄存器

GPIO控制器相关资源和控制方法我们在GPIO驱动[第二节](https://github.com/mdxz2048/Linux-driver-development-basics/blob/main/ji-chu-pian/er-gpio-qu-dong/2imx6ul-de-gpio-xiang-guan-ji-cun-qi.md#2-imx6ul%E7%9A%84gpio%E7%9B%B8%E5%85%B3%E5%AF%84%E5%AD%98%E5%99%A8)进行了介绍，数据都来自《[IMX6ULLRM.pdf](https://www.nxp.com/docs/en/reference-manual/IMX6ULLRM.pdf)》。

我们这里再对GPIO的配置梳理一遍：

### 设置时钟源

1. 通过CCM(Clock Controller Module)的CCGR( Clock Gating Register)控制器来设置时钟源,其中对应关系如下：

    | GPIO  | 寄存器名称  | 寄存器地址 | 描述             |
    | ----- | ----------- | ---------- | ---------------- |
    | GPIO4 | CCGR3[CG6]  | 20C_4074   | GPIO4_CLK_ENABLE |
    | GPIO5 | CCGR1[CG15] | 20C_406C   | GPIO5_CLK_ENABLE |

    

2. CCGR寄存器每组(2bit)的设置值和功能对应关系如下:

    | CGR value | Clock Activity Description                                   |      |
    | --------- | ------------------------------------------------------------ | ---- |
    | 00        | Clock is off during all modes. Stop enter hardware handshake is disabled. |      |
    | 01        | Clock is on in run mode, but off in WAIT and STOP modes      |      |
    | 10        | Not applicable (Reserved).                                   |      |
    | 11        | Clock is on during all modes, except STOP mode.              |      |

    

    

2. 因此，我们要操作(`open`)对应的按键时，要把对应的CCGR寄存器的位设置为0x11,就可以为GPIO模块提供时钟源;

### 设置为GPIO模式

1. `IOMUXC_SW_MUX_CTL_PAD_NAND_CE1_B`的`[0,3]`共4个位来设置`GPIO4_14`工作在GPIO模式,设置为`0101`;

    ![image-20220409135706279](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409135706279.png)

2. `IOMUXC_SNVS_SW_MUX_CTL_PAD_SNVS_TAMPER1`的的`[0,3]`共4个位来设置`GPIO5_01`工作在GPIO模式,应设置为`0101`;

    ![image-20220409140210120](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409140210120.png)

3. 至此，设置`GPIO4_IO14`、`GPIO5_IO01`引脚工作在GPIO模式下，设置对应关系如下:

    | 引脚       | 对应寄存器名称和操作位                       | 寄存器地址 | 目标值 |
    | ---------- | -------------------------------------------- | ---------- | ------ |
    | GPIO4_IO14 | IOMUXC_SW_MUX_CTL_PAD_NAND_CE1_B             | 20E01B0h   | 0101   |
    | GPIO5_IO01 | IOMUXC_SNVS_SW_MUX_CTL_PAD_SNVS_TAMPER1[0,3] | 229000Ch   | 101    |

### GPIO内部寄存器的操作

这里我们需要通过GPIO控制器将引脚设置为输入模式，然后读取对应的引脚状态寄存器来读取对应引脚状态，以此来判断按键是否按下。

1. 每组GPIO由对应一组寄存器控制，每组地址如下图。

    ![image-20220409141609134](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409141609134.png)

2. 每组寄存器包含了如下8组寄存器,这里以GPIO4为例展示，具体的寄存器定义参考《[2-imx6ul的GPIO相关寄存器](https://github.com/mdxz2048/Linux-driver-development-basics/blob/main/ji-chu-pian/er-gpio-qu-dong/2imx6ul-de-gpio-xiang-guan-ji-cun-qi.md#2-imx6ul%E7%9A%84gpio%E7%9B%B8%E5%85%B3%E5%AF%84%E5%AD%98%E5%99%A8)》

    ![image-20220409141857966](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409141857966.png)

3. 接下来配置方向寄存器为输入方向,即把GPIO4_14、GPIO5_01的对应位设置为`0`;

    ![image-20220409142523466](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409142523466.png)

4. 设置完成后，就可以通过引脚状态寄存器来读取引脚的状态来判断按键是否被按下:

    ![image-20220409143102444](https://raw.githubusercontent.com/mdxz2048/mddxz_top_img/main/image-20220409143102444.png)

5. 这里我们整理一下，GPIO4_14、GPIO5_01对应的GPIO地址如下

    | GPIO       | GPIO寄存器组起始地址 | GPIOx_DR(IN) | GPIOx_PSR（OUT） |
    | ---------- | -------------------- | ------------ | ---------------- |
    | GPIO4_IO14 | 0x020A8000           | 0x020A8000   | 20A_8008         |
    | GPIO5_IO01 | 0x020AC000           | 0x020AC000   | 20A_C008         |

6. 至此，GPIO相关寄存器就配置完成了。

## 驱动程序

