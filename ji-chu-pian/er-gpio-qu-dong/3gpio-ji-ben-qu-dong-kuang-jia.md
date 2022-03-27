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

驱动框架及测试代码，请点击[这里](https://github.com/mdxz2048/Linux-driver-development-basics-code/blob/main/1_HelloDrv/hello_drv.c)查看。

我们这里的驱动框架是一个精简版本，只是先了LED的打开功能，主要演示LED的基本操作，更完整的版本后续~~补充。~~

### 代码解释

#### [led_drv.c](https://github.com/mdxz2048/Linux-driver-development-basics-code/blob/main/2_GPIO/led_drv.c)

1. 确定主设备号；

   ```c
   /*1. 确定主设备号；*/
   static int led_major; /* default to dynamic major */
   static struct class *led_class;
   ```

   

2. 定义`file_operations`结构体；

   ```c
   /*2. 定义自己的file_operations结构体；*/
   /*
    * file operations
    */
   static const struct file_operations led_ops = {
   	.owner = THIS_MODULE,
   	.open = led_open,
   	.write = led_write,
   };
   ```

   

3. 实现对应的`open/read/write`等函数，填入`file_operations`结构体。我们这里只实现了open和write功能；

   ```c
   static int led_open(struct inode *inode, struct file *filp)
   {
   	printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
   	/*CONFIG GPIO */
   	*SW_MUX_CTL_PAD_SNVS_TAMPER3 &= ~0xf; // clear
   	*SW_MUX_CTL_PAD_SNVS_TAMPER3 |= 0x5;  // set
   
   	/*CONFIG GPIO OUTPUT*/
   	*GPIO5_GDIR |= (1 << 3);
   	return 0;
   }
   
   static int led_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
   {
   	char value = 0;
   	int ret = -1;
   	printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
   
   	ret = copy_from_user(&value, buf, 1);
   	if (value == 0)
   	{
   		// set GPIO to LED OFF
   		*GPIO5_DR |= (1 << 3); // set 3 bit
   	}
   	else
   	{
   		// set GPIO to LED ON
   		*GPIO5_DR &= ~(1 << 3); // clear 3 bit
   	}
   	return 0;
   }
   ```

   

4. 入口函数：安装驱动时，就会去调用入口函数；

   ```c
   static int __init led_init(void)
   {
   	int i = 0;
   	printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
   	/*ioremap*/
   	SW_MUX_CTL_PAD_SNVS_TAMPER3 = ioremap(0x2290014, 4);
   	GPIO5_GDIR = ioremap(0x20AC004, 4);
   	GPIO5_DR = ioremap(0x20AC000, 4);
   	/*class_create*/
   	led_class = class_create(THIS_MODULE, "mdxz_led");
   	if (IS_ERR(led_class))
   		return PTR_ERR(led_class);
   	led_major = register_chrdev(0, DEVNAME, &led_ops);
   	if (led_major < 0)
   	{
   		printk(DEVNAME
   			   ": could not get major number\n");
   		class_destroy(led_class);
   		return led_major;
   	}
   
   	device_create(led_class, NULL, MKDEV(led_major, 0), NULL, "mdxz_led%d", i);
   
   	return 0;
   }
   ```

   

5. 出口函数：卸载驱动时，就会去调用出口函数；

   ```c
   static void __exit led_exit(void)
   {
   	iounmap(SW_MUX_CTL_PAD_SNVS_TAMPER3);
   	iounmap(GPIO5_GDIR);
   	iounmap(GPIO5_DR);
   	unregister_chrdev(led_major, DEVNAME);
   	class_destroy(led_class);
   	device_destroy(led_class, MKDEV(led_major, 0));
   }
   ```

   

6. 注册入口函数

   ```
   module_init(led_init);
   module_exit(led_exit);
   MODULE_LICENSE("GPL");
   ```

### 实验测试

#### 编译过程

#### 实验过程

### 补充内容

1. 我们这里的示例代码中，使用了模块动态加载的方法方便测试，内核模块还可以通过静态加载的方式编译进内核，关于两者的区别可以参考这篇博客[《Linux内核模块分析（module_init宏》](https://blog.csdn.net/lu_embedded/article/details/51432616)；

