# 4-LED驱动(imx6ul)

## LED驱动框架设计

### GPIO驱动基本框架的缺点

[3-GPIO基本驱动框架](3gpio-ji-ben-qu-dong-kuang-jia.md)章节里的GPIO框架，是一个最简陋的驱动框架，主要有以下问题：

1. led_open()、led_write()接口直接操作了GPIO5_03的寄存器，可移植性差，无法适配其他GPIO接口，更无法适配其他单板；
2. 每次只能操作1个GPIO引脚，驱动的灵活度较低。

### LED驱动框架介绍

为了适配多个开发板(CPU)、支持多个GPIO口的操作，我们对GPIO基本框架做出如下调整。

- 将GPIO的操作流程抽象，形成了与硬件无关的`led_drv.c`，包含led_drv_open()、led_drv_write()、led_drv_close()等函数。这些函数中在具体单板中，去调用具体单板的实际操作接口；
- 对于具体单板的操作，抽象到`100_ask_imx6ull_led.c`，这里主要包括了led_opr_init()、led_opr_ctl()、led_opr_exit()等实际的GPIO操作函数，根据led_drv.c中传来的参数来选择单板对应的GPIO引脚进行相应操作。

## 程序讲解

### 单板操作100_ask_imx6ull_led.c

本节代码，可以在[这里](https://github.com/mdxz2048/Linux-driver-development-basics-code/blob/main/2_GPIO/2_4_led_driver_imx6ull/100_ask_imx6ull_led.c)查看。

#### 功能梳理

单板操作是我们驱动框架中实际和硬件相关的操作，这里具体来说就是我们实际对LED的操作。主要做的工作我们GPIO驱动框架章节基本一致，主要包括以下

- 映射相关寄存器的地址；
- 配置相关引脚为GPIO功能；
- 通过设置输入、输出来实现LED亮灭；
- 实现exit程序，卸载驱动时，释放相关资源；

#### 代码

##### 创建led单板操作对象

新建一个`led_operations`格式对象，将单板支持的LED具体操作抽象到这个对象中，在后边的硬件无关的驱动程序中，只要获取这个`led_operations`对象就可以操作具体的单板led。

```c
typedef struct
{
	int (*init)(void);		/* 初始化LED, which-哪个LED */
	int (*ctl)(int status); /* 控制LED, which-哪个LED, status:1-亮,0-灭 */
	int (*exit)(void);
} led_operations_t;
```

##### 初始化

led_opr_init函数主要是初始化GPIO的状态，映射相关资源；

```c
static int led_opr_init(void)
{
    printk("%s %s line %d, \n", __FILE__, __FUNCTION__, __LINE__);
    // operations code
    /*IOREMAP*/
    if (!CCM_CCGR1)
    {
        CCM_CCGR1                   = ioremap(0x20C406C, 4);
        SW_MUX_CTL_PAD_SNVS_TAMPER3 = ioremap(0x2290014, 4);
        GPIO5_GDIR                  = ioremap(0x20AC004, 4);
        GPIO5_DR                    = ioremap(0x20AC000, 4);
    }
    /*CONFIG GPIO */
    *SW_MUX_CTL_PAD_SNVS_TAMPER3 & = ~0xf;  // clear
    *SW_MUX_CTL_PAD_SNVS_TAMPER3 | = 0x5;   // set
    /*CONFIG GPIO OUTPUT*/
    *GPIO5_GDIR | = (1 << 3);

    return 0;
}
```

##### exit程序释放资源

```c
static int led_opr_exit(void)
{
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    // operations code
    /*IOUNMAP*/
    iounmap(CCM_CCGR1);
    iounmap(SW_MUX_CTL_PAD_SNVS_TAMPER3);
    iounmap(GPIO5_GDIR);
    iounmap(GPIO5_DR);

    return 0;
}
```

##### LED控制接口

led_opr_ctl函数是实现控制GPIO的输出的高、低，以实现具体led的亮灭。

```c
static int led_opr_ctl(int status)
{
    printk("%s %s line %d, %s\n", __FILE__, __FUNCTION__, __LINE__, status ? "on" : "off");
    // operations code
    if (status == 0)
    {
        // set GPIO to LED OFF
        *GPIO5_DR | = (1 << 3);  // set 3 bit
    }
    else
    {
        // set GPIO to LED ON
        *GPIO5_DR & = ~(1 << 3);  // clear 3 bit
    }

    return 0;
}
```

##### 获取资源地址

get_board_led_opr是被驱动调用的，用来获取led_opration的操作对象；

```c
led_operations_t *get_board_led_opr(void)
{
    led_opr_100ask.init = led_opr_init;
    led_opr_100ask.ctl  = led_opr_ctl;
    led_opr_100ask.exit = led_opr_exit;

    return &led_opr_100ask;
}
```



### 硬件无关led_drv.c

本节代码，可以在[这里](https://github.com/mdxz2048/Linux-driver-development-basics-code/blob/main/2_GPIO/2_4_led_driver_imx6ull/led_drv.c)查看。

#### 功能梳理

这个驱动被Linux内核直接调用，我们需要在这个驱动里实现应用层可能要用到的open()、write()、close()函数与之对应的led_open()、led_write()、led_close()等。

- 确定主设备号(`led_major`)；
- 在内核中创建对象(`led_class`)；
- 实现init()、open()、write()、exit()接口，并注册到`file_operations`结构体**led_ops**；

#### 代码

##### 入口函数led_init的实现与注册

初始化函数是在insmod的时候会被调用，主要完成以下任务：

1. 创建`led_class`对象；
2. 注册字符型设备到内核，获得主设备号`led_major`；
3. 创建设备；
4. 通过`get_board_led_opr()`获取led的操作接口地址，这里主要是为后续的open()、write()函数中具体单板操作函数提供具体的接口地址；

```c
static int __init led_init(void)
{
    int i = 0;
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    /*class_create*/
    led_class = class_create(THIS_MODULE, "mdxz_led");
    if (IS_ERR(led_class))
        return PTR_ERR(led_class);
    /*get p_led_opr*/
    p_led_opr = get_board_led_opr();

    led_major = register_chrdev(0, DEVNAME, &led_ops);
    if (led_major < 0)
    {
        printk(DEVNAME ": could not get major number\n");
        class_destroy(led_class);
        return led_major;
    }

    device_create(led_class, NULL, MKDEV(led_major, 0), NULL, "mdxz_led%d", i);

    return 0;
}
module_init(led_init);
```

##### 出口函数led_exit的实现与注册

出口函数主要是做和入口函数反向的事情，释放掉相关资源、去注册设备、销毁`led_class`。

```c
static void __exit led_exit(void)
{
    struct inode *inode = file_inode(file);
    int minor = iminor(inode);
    p_led_opr->exit(minor);
    unregister_chrdev(led_major, DEVNAME);
    class_destroy(led_class);
    device_destroy(led_class, MKDEV(led_major, 0));
}
module_exit(led_exit);
```

##### open、write和close接口

led_drv_open主要是调用单板的具体的初始化函数(`p_led_opr->init()`)来初始化GPIO相关功能(配置IOMUXC、配置GPIO引脚等)；

```c
static int led_drv_open(struct inode *inode, struct file *filp)
{
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    p_led_opr->init();

    return 0;
}
```

led_drv_write主要是将用户态的buf读进来，通过调用实际的单板控制函数 `p_led_opr->ctl(minor, value)`并传入参数，来控制实际的LED亮灭;

```c
static int led_drv_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
    char value = 0;
    int  ret   = -1;
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);

    ret = copy_from_user(&value, buf, 1);

    p_led_opr->ctl(value);

    return 0;
}
```

led_drv_close这里默认不做动作，因为我们默认为关掉led后，这里没有具体操作。当然，可以添加led-off相关代码，关掉led。

```c
static int led_drv_close(struct inode *node, struct file *file)
{
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    /*提取次设备号*/

    return 0;
}
```

至此，我们主要通过3个函数来填充`file_operations`这个结构体，实现LED的驱动层；

```c
static const struct file_operations led_ops = {
    .owner   = THIS_MODULE,
    .open    = led_drv_open,
    .write   = led_drv_write,
    .release = led_drv_close,
};
```

