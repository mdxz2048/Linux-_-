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

可以通过 进入`映射表the mapping table`来增加配置来对`Pin configuration`进行编程。可以参考下面*Board/machine configuration*部分。

配置参数的格式和含义(`PLATFORM_X_PULL_UP` )完全由pin控制器驱动来定义。

引脚配置器驱动实现回调，用于引脚控制器操作中更改引脚配置。如下所示：

```c
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/pinconf.h>
#include "platform_x_pindefs.h"

static int foo_pin_config_get(struct pinctrl_dev *pctldev,
                unsigned offset,
                unsigned long *config)
{
        struct my_conftype conf;

        ... Find setting for pin @ offset ...

        *config = (unsigned long) conf;
}

static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                unsigned offset,
                unsigned long config)
{
        struct my_conftype *conf = (struct my_conftype *) config;

        switch (conf) {
                case PLATFORM_X_PULL_UP:
                ...
                }
        }
}

static int foo_pin_config_group_get (struct pinctrl_dev *pctldev,
                unsigned selector,
                unsigned long *config)
{
        ...
}

static int foo_pin_config_group_set (struct pinctrl_dev *pctldev,
                unsigned selector,
                unsigned long config)
{
        ...
}

static struct pinconf_ops foo_pconf_ops = {
        .pin_config_get = foo_pin_config_get,
        .pin_config_set = foo_pin_config_set,
        .pin_config_group_get = foo_pin_config_group_get,
        .pin_config_group_set = foo_pin_config_group_set,
};

/* Pin config operations are handled by some pin controller */
static struct pinctrl_desc foo_desc = {
        ...
        .confops = &foo_pconf_ops,
};
```

由于某些控制器拥有特殊的逻辑来处理整组引脚，因此它们可以拥有特殊的整组引脚控制功能。对于不想处理的组，或者只是在进行组级别的处理然后迭代配置到所有引脚，则允许`pin_config_group_set()`返回错误代码`-EAGAIN`，在这种情况下，每组单独的Pin也将由单独的pin_config_set()调用处理。



## [Interaction with the GPIO subsystem配合GPIO子系统](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html#interaction-with-the-gpio-subsystem)

GPIO驱动程序可能希望在同样注册为引脚控制器引脚的同一组物理引脚上执行各种类型的操作。

首先，这两个子系统可以用作完全正交的子系统，详细信息参考“pin control requests from drivers”和“drivers needing both pin control and GPIOs”部分。但在某些情况下，引脚和GPIO之间需要跨子系统映射。

由于引脚控制器子系统具有印尼叫控制器本地的引脚空间，因此我们需要一个映射，以便引脚控制子系统可以确定哪个引脚控制器处理某个GPIO引脚的控制。由于单个引脚控制器可以多路复用多个GPIO引脚范围(通常是具有一组引脚的SoC，但在内部有几个GPIO硅块，每个芯片都模块化为一个结构`gpio_chip`)，因此可以将任意数量的GPIO范围添加到引脚控制器实例，如下所示:

```c
struct gpio_chip chip_a;
struct gpio_chip chip_b;

static struct pinctrl_gpio_range gpio_range_a = {
        .name = "chip a",
        .id = 0,
        .base = 32,
        .pin_base = 32,
        .npins = 16,
        .gc = &chip_a;
};

static struct pinctrl_gpio_range gpio_range_b = {
        .name = "chip b",
        .id = 0,
        .base = 48,
        .pin_base = 64,
        .npins = 8,
        .gc = &chip_b;
};

{
        struct pinctrl_dev *pctl;
        ...
        pinctrl_add_gpio_range(pctl, &gpio_range_a);
        pinctrl_add_gpio_range(pctl, &gpio_range_b);
}
```

因此，这个复杂系统具有1个引脚控制器可以处理两个不同的GPIO芯片。“chip a”有16个引脚，“chip b”有8个引脚。“chip a”和“chip b”具有不同的`.pin_base`，这意味着GPIO范围的起始引脚号。

“chip a”的GPIO范围从GPIO基地址32开始，实际引脚范围也从32开始。然后，“chip b”对于GPIO范围和引脚范围有着不同的其实偏移量。“chip b”从GPIO编号48开始，引脚范围从64开始。

我们可以使用`pin_base`将gpio number转化为实际的引脚号。它们被映射在全局的GPIO引脚空间。

**chip a:**

- GPIO range : [32 .. 47]
- pin range : [32 .. 47]

**chip b:**

- GPIO range : [48 .. 55]
- pin range : [64 .. 71]



上面的示例假设GPIO和引脚之间的映射是线性的。如果映射是稀疏或者随即的，则可以在如下范围内编码任意引脚号数组。

```c
static const unsigned range_pins[] = { 14, 1, 22, 17, 10, 8, 6, 2 };

static struct pinctrl_gpio_range gpio_range = {
        .name = "chip",
        .id = 0,
        .base = 32,
        .pins = &range_pins,
        .npins = ARRAY_SIZE(range_pins),
        .gc = &chip;
};
```

在这种情况下`pin_base` 属性将被忽略。如果引脚组的名字已知，则可以使用`pinctrl_get_group_pins()`初始化上述结构体和`npins`元素。例如对于引脚组“foo”：

```c
pinctrl_get_group_pins(pctl, "foo", &gpio_range.pins,
                       &gpio_range.npins);
```

当调用引脚控制子系统(pin control subsystem)的GPIO特定功能时，这些范围将被用于通过检查查引脚并将其与所有控制器上的引脚范围进行匹配来查找相应的引脚控制器。当找到处理匹配范围的引脚控制器时，将在该特定引脚控制器上调用特定于GPIO的功能。

对于处理引脚偏置、引脚多路复用等的所用功能，引脚控制子系统将从从传入的GPIO编号中查找相应的引脚号，并使用range内检索引脚号。之后，引脚控制子系统将其传递给引脚控制器驱动，驱动程序将在其处理的编号范围内获得引脚号。此外，它还传递了范围的ID值，以便引脚控制器知道它应该处理哪个range。

不推荐从pinctrl 驱动中调用`pinctrl_add_gpio_range`。请从参考文档*Documentation/devicetree/bindings/gpio/gpio.txt*了解如何绑定`Pinctrl`和`gpio驱动`。

## PINMUX interfaces引脚复用接口

调用的时候使用前缀`pinmux_*`。任何其他调用都不应该使用该前缀。



## What is pinmuxing?

PINMUX，也叫padmux、ballmux、alternate functions(备用功能)或mission modes(任务模式)，时芯片供应商生产某种电气封装的一种方式，多用于相互排斥(矛盾)的功能，实际用哪个功能取决于应用。在这种情况下，我们所说的”应用”是指一种将封装焊接或连接到电子系统的方法，即使是框架也可以在运行时更改功能。

以下是从下面看到的PGA（引脚栅格阵列）芯片的示例：

```c
     A   B   C   D   E   F   G   H
   +---+
8  | o | o   o   o   o   o   o   o
   |   |
7  | o | o   o   o   o   o   o   o
   |   |
6  | o | o   o   o   o   o   o   o
   +---+---+
5  | o | o | o   o   o   o   o   o
   +---+---+               +---+
4    o   o   o   o   o   o | o | o
                           |   |
3    o   o   o   o   o   o | o | o
                           |   |
2    o   o   o   o   o   o | o | o
   +-------+-------+-------+---+---+
1  | o   o | o   o | o   o | o | o |
   +-------+-------+-------+---+---+
```

这不是俄罗斯方块，要想到的是国际象棋。并非所有的PGA/BGA封装都是棋盘式的，大封装根据不同的设计模式在某些排列方式上有“孔”，但我们用这个做一个简单的例子。在看到的因教宗，一些将被VCC、GND占用，以便为芯片供电，并且相当多的引脚被外部存储器接口等大型端口占用。其余引脚通常需要进行引脚多路复用。

上边示例8x8 PGA封装将其物理引脚分配引脚号为0~63.它将使用 `pinctrl_register_pins()`命名引脚 { A1, A2, A3 ... H6, H7, H8 }和一个舒适的数据集合。

在这个8x8 PGA封装中{ A8, A7, A6, A5 }可以被用作SPI端口(4个引脚CLK, RXD, TXD, FRM)。在这种情况下,引脚B5可以被用作GPIO引脚。然而，在其他的设置中，{ A5, B5 }可以被用作I2C引脚(只占用两个引脚SCL, SDA)。不必说，我们不能同时使用I2C端口和SPI端口。然而，在封装内部，执行SPI逻辑的部分可以通过引脚{ G4, G3, G2, G1 }路由出去。

在 { A1, B1, C1, D1, E1, F1, G1, H1 }的这一排，我们有一些特殊的东西，它是一个外部MMC总线，我们可以有2、4或8位宽，将分别占用2、4或8个引脚，因此要采用{ A1, B1 }、{ A1, B1, C1, D1 }或者所有有引脚。如果我们使用8位，我们当然不能在{ G4, G3, G2, G1 }引脚上使用SPI端口。

通过这种方式，芯片内部存在的硅块可以在不同的引脚范围内进行多路复用。经常，同一代SoC会包含多个I2C、SPI、SDIO/MMC，硅快可以通过pinmux设置将它们路由到不同的引脚。

由于GPIO引脚总是短缺，因此，如果某些其他I/O端口当前未使用，则通常能够使用几乎任何引脚作为GPIO引脚。

## Pinmux conventions PINMUX约定

Pinmux功能在引脚控制子系统中的目的是抽象pinmux设置，并将其提供给您在机器配置中进行实例化。它相关的控制有CLK、GPIO和稳压子系统，因此设备将通过其请求复用器的设置，也可以为GPIO请求设置单个引脚。

### 定义

- FUNCTION 可以通过驻留内核文件drivers/pinctrl/*的引脚控制子系统来切换输入和输出；例如，有3个pinmux函数，一个用于SPI，一个用于I2C，一个用于MMC；

- FUNCTION 假设一维数组的枚举从0开始。这种情况下数组可以是这样 { spi0, i2c0, mmc0 }为三个有效的函数；

- FUNCTION 具有generic level定义的PIN GROUPS---因此某个功能与一组特定的引脚关联，可能是多个，也可能是单个引脚。例如I2C功能与引脚 { A5, B5 }关联，再控制器的pin space枚举{ 24, 25 }；

  SPI功能与引脚组{ A8, A7, A6, A5 } 和 { G4, G3, G2, G1 }关联，那么枚举各自的引脚 { 0, 8, 16, 24 } 和{ 38, 46, 54, 62 }；

  每个引脚控制器的Group name必须是唯一的，同一个控制器没有两个groups有同样的名字；

- 对于一套引脚，FUNCTION和PIN GROUP结合起来确定了一个特定的功能。FUNCTION和PIN GROUPS机器特定的机器细节被保存在PINMUX驱动内，从外部只知道枚举，并且驱动器可以请求：

  - 具有特定选择器的FUNCTION（>=0）；
  - 与特定FUNCTION关联的组的列表；
  - 该列表中的某个组要为某个功能激活；

  如上所述，引脚组是子描述的，因此内核将从驱动程序中检索特定组中的实际引脚范围。

- 某个PIN CONTROOLER上的FUNCTION和GROUPS通过板级文件、设备树和类似的机器设置配置机制映射到某个设备，类似于条机器如何连接到设备，通常按照名称。定义引脚控制器（pin controller）、功能(function)和group，从而标定唯一设备哟啊是用的引脚。

  例如：

  ```
  {
          {"map-spi0", spi0, pinctrl0, fspi0, gspi0},
          {"map-i2c0", i2c0, pinctrl0, fi2c0, gi2c0}
  }
  ```

  必须为每个映射北奥分配一个状态名称、引脚控制器、设备和功能。group部署必须的，如果省略，将选择驱动器提供的设用于该function的第一个组，这对于简单情况很适用。

  可以将多个group映射到设备，引脚控制器和function的相同组合。这适用于特定引脚控制器上的某个功能可能在不同配置中使用不同引脚集的情况。

- 在某个PIN控制器上使用某个PIN GROUP的某个功能的PIN，采用先到先得的方式。因此，如果某些其他设备多路复用器设置或GPIO引脚请求已经占用了该物理引脚，您将被拒绝使用它。要获取(激活)新设置，必须先释放(停用)旧设置。

有时，文档和硬件寄存器将围绕pad而不是引脚，可能与封装内的引脚数量匹配，也可能不匹配。因此，选择对你有意义的枚举，仅为可以控制的引脚定义枚举。
