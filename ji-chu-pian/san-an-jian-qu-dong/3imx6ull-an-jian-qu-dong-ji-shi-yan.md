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

1. 通过CCM控制器(Clock Controller Module)来设置时钟源;
2. 



## 驱动程序

