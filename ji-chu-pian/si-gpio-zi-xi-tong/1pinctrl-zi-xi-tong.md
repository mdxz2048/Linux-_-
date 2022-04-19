# 1-PINCTRL子系统

 本文档概述Linux系统中的PINCTRL子系统，全称为pin control subsystem，翻译自[PINCTRL (PIN CONTROL) subsystem](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html)

PINCTRL子系统处理如下：

- 列举、命名控制引脚：

- 复用引脚，细节见下文;

- 配置一脚，例如软件控制偏移、驱动模式特定引脚(例如上拉、下拉，开漏、负载电容等)

    

## [Top-level interface顶层接口](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html#top-level-interface)

**PIN CTNTROLLER**定义：

- 引脚控制器是一块硬件。通常是一组控制引脚的寄存器。它可以特定的引脚配置 复用(引脚)、偏移量、负载电容、设置驱动强度等。

**PIN**定义：

- PINS等效于pads、fingers、ball或任何要包装成输入或者输出行(line)，这些行(line)由一个unsigned int来控制，范围是0~最大引脚数。这个unsigned int的数对应了系统中的每个PIN控制器，因此，系统中可能有很多个这种unsigned int的数。此引脚空间可能比较稀疏，即空间中可能存在没有引脚的数字间隔。(实际引脚数小于unsigned int对应的32位数)。

当一个引脚控制器(PIN CONTROLLER)被初始化，它将注册一个描述符到pin control框架，并且这个描述符包含了一个引脚描述器的数组，这些引脚描述器描述了某个引脚对应的特定的引脚控制器。

例如这里有一个PGA（Pin Grid Array），引脚矩阵

```sh
     A   B   C   D   E   F   G   H

8    o   o   o   o   o   o   o   o

7    o   o   o   o   o   o   o   o

6    o   o   o   o   o   o   o   o

5    o   o   o   o   o   o   o   o

4    o   o   o   o   o   o   o   o

3    o   o   o   o   o   o   o   o

2    o   o   o   o   o   o   o   o

1    o   o   o   o   o   o   o   o
```



如何注册一个引脚控制器(PIN CONTROLLER)，命名所有的引脚。我们可以在驱动中参考 下边代码。

```c
#include <linux/pinctrl/pinctrl.h>

const struct pinctrl_pin_desc foo_pins[] = {
        PINCTRL_PIN(0, "A8"),
        PINCTRL_PIN(1, "B8"),
        PINCTRL_PIN(2, "C8"),
        ...
        PINCTRL_PIN(61, "F1"),
        PINCTRL_PIN(62, "G1"),
        PINCTRL_PIN(63, "H1"),
};

static struct pinctrl_desc foo_desc = {
        .name = "foo",
        .pins = foo_pins,
        .npins = ARRAY_SIZE(foo_pins),
        .owner = THIS_MODULE,
};

int __init foo_probe(void)
{
        int error;

        struct pinctrl_dev *pctl;

        error = pinctrl_register_and_init(&foo_desc, <PARENT>,
                                          NULL, &pctl);
        if (error)
                return error;

        return pinctrl_enable(pctl);
}
```

要为PINMUX、PINCONF以及选择的驱动程序使能pintctrl subsystem,你需要从机器的Kconfig条目中选择它们，i那位它们与使用它们的机器集成比较紧密。可以参考[arch/arm/mach-u300/Kconfig](arch/arm/mach-u300/Kconfig)的示例。

Pins通常有更花哨的名字。你可以在你的芯片(Soc)数据手册中找到，[pinctrl.h](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinctrl.h)文件提供了一个宏定义PINCTRL_PIN()增加结构体成员,定义和引用示例如下:

```c
/* Convenience macro to define a single named or anonymous pin descriptor */
#define PINCTRL_PIN(a, b) { .number = a, .name = b }
#define PINCTRL_PIN_ANON(a) { .number = a }
/*example*/
const struct pinctrl_pin_desc foo_pins[] = {
- PINCTRL_PIN(0, "A1"),
- PINCTRL_PIN(1, "A2"),
- PINCTRL_PIN(2, "A3"),
+ PINCTRL_PIN(0, "A8"),
+ PINCTRL_PIN(1, "B8"),
+ PINCTRL_PIN(2, "C8"),
- PINCTRL_PIN(61, "H6"),
- PINCTRL_PIN(62, "H7"),
- PINCTRL_PIN(63, "H8"),
+ PINCTRL_PIN(61, "F1"),
+ PINCTRL_PIN(62, "G1"),
+ PINCTRL_PIN(63, "H1"),
};
```

 正如你所见，我枚举了从左上角的0到右下角的63的引脚。这个枚举是任意选择的，再实践中，你需要仔细考虑编号系统，以便它与驱动中的寄存器布局相匹配，否则代码将变得复杂。还必须考虑将偏移量与可能由引脚控制器处理的 GPIO 范围相匹配。

对于有467个焊盘(而不是实际的引脚),我使用这种方式枚举。像这样，再芯片的周围进行定义，这似乎也是行业标准(这些焊盘是有对应名称的)。

```
  0 ..... 104
466        105
  .        .
  .        .
358        224
 357 .... 225
```

## [Pin groups](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html#pin-groups)

一些控制需要处理一组引脚，因此引脚控制器子系统(pin controller subsystem)有一个机制来枚举引脚组，并检索属于特定组的实际枚举引脚。

例如，有一组SPI接口的引脚是 { 0, 8, 16, 24 }, 另一组I2C引脚是{ 24, 25 }.

这两组一般在引脚控制器pinctrl_ops 中表现为:

```c
#include <linux/pinctrl/pinctrl.h>

struct foo_group {
        const char *name;
        const unsigned int *pins;
        const unsigned num_pins;
};

static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
static const unsigned int i2c0_pins[] = { 24, 25 };

static const struct foo_group foo_groups[] = {
        {
                .name = "spi0_grp",
                .pins = spi0_pins,
                .num_pins = ARRAY_SIZE(spi0_pins),
        },
        {
                .name = "i2c0_grp",
                .pins = i2c0_pins,
                .num_pins = ARRAY_SIZE(i2c0_pins),
        },
};


static int foo_get_groups_count(struct pinctrl_dev *pctldev)
{
        return ARRAY_SIZE(foo_groups);
}

static const char *foo_get_group_name(struct pinctrl_dev *pctldev,
                                unsigned selector)
{
        return foo_groups[selector].name;
}

static int foo_get_group_pins(struct pinctrl_dev *pctldev, unsigned selector,
                        const unsigned **pins,
                        unsigned *num_pins)
{
        *pins = (unsigned *) foo_groups[selector].pins;
        *num_pins = foo_groups[selector].num_pins;
        return 0;
}

static struct pinctrl_ops foo_pctrl_ops = {
        .get_groups_count = foo_get_groups_count,
        .get_group_name = foo_get_group_name,
        .get_group_pins = foo_get_group_pins,
};


static struct pinctrl_desc foo_desc = {
...
.pctlops = &foo_pctrl_ops,
};
```

引脚控制子系统(pin control subsystem)通过调用`.get_groups_count()`来确定合法引脚的总数，并通过其他函数来检索组的名字和引脚。实际的组的数据结构数据保持取决于驱动层，这只是一个简单的示例-实际上你可能需要在结构体中加入更多成员，例如每组关联的特定寄存器的范围等。

## [Pin configuration](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html#pin-configuration)

引脚可以被软件以各种而样的方式配置，更多的与其被用作输入或者输出时的电子特性有关。例如，你可能会配置引脚是高阻态/三态，来让引脚断开连接。

