# 6-基于设备树的LED驱动实验

# 实验

## 编译

### 编译设备树文件

1. 进入开发板内核

    ```sh
    cd /home/mdxz/lzp/Embeded/100ask_imx6ull-sdk/Linux-4.9.88
    ```

    

2. 将我们的设备树节点内容添加到开发板的设备树文件中，imx6ull_pro开发板对应的的设备树文件位于如下目录：

    ```sh
    /home/mdxz/lzp/Embeded/100ask_imx6ull-sdk/Linux-4.9.88/arch/arm/boot/dts/100ask_imx6ull-14x14.dts
    ```

3. 使用`vim`打开开发板设备树文件

    ```sh
    {
        model = "Freescale i.MX6 ULL 14x14 EVK Board";
        compatible = "fsl,imx6ull-14x14-evk", "fsl,imx6ull";
        chosen {
            stdout-path = &uart1;
        };
    
        memory {
            reg = <0x80000000 0x20000000>;
        };
    
        reserved-memory {
            #address-cells = <1>;
            #size-cells = <1>;
            ranges;
    
            linux,cma {
                compatible = "shared-dma-pool";
                reusable;
                size = <0x14000000>;
                linux,cma-default;
            };
        };
    
        backlight {
            compatible = "pwm-backlight";
            pwms = <&pwm1 0 1000>;
            brightness-levels = <0 1 2 3 4 5 6 8 10>;
            default-brightness-level = <8>;
            status = "okay";
        };
    
        pxp_v4l2 {
            compatible = "fsl,imx6ul-pxp-v4l2", "fsl,imx6sx-pxp-v4l2", "fsl,imx6sl-pxp-v4l2";
            status = "okay";
        };
        ..........
    }
    ```

    

4. 将我们修改的节点添加进来

    ```sh
    #include <dt-bindings/input/input.h>
    #include "imx6ull.dtsi"
    #define GROUP_PIN(g,p) ((g<<16) | (p))
    {
        model = "Freescale i.MX6 ULL 14x14 EVK Board";
        compatible = "fsl,imx6ull-14x14-evk", "fsl,imx6ull";
    
            mdxz_led@0 {
                    compatible = "mdxz,leddrv";
                    pin = <GROUP_PIN(3, 1)>;
            };
    
            mdxz_led@1 {
                    compatible = "mdxz,leddrv";
                    pin = <GROUP_PIN(5, 8)>;
            };
    
        chosen {
            stdout-path = &uart1;
        };
    
    
    
    ```

    

5. 这里注意需要加上宏定义`#define GROUP_PIN(g,p) ((g<<16) | (p))`，然后保存退出；

6. 在内核根目录下执行**make dtbs**来编译设备树文件；

    ```sh
    # mdxz @ ubuntu in ~/lzp/Embeded/100ask_imx6ull-sdk/Linux-4.9.88 on git:d0b34fd9c024 x [20:33:14] 
    $ make dtbs                                                                                            
      CHK     include/config/kernel.release
      CHK     include/generated/uapi/linux/version.h
      CHK     include/generated/utsrelease.h
      CHK     include/generated/bounds.h
      CHK     include/generated/timeconst.h
      CHK     include/generated/asm-offsets.h
      CALL    scripts/checksyscalls.sh
      DTC     arch/arm/boot/dts/100ask_imx6ull-14x14.dtb
    ```

    

7. 然后将编译出来的设备树文件存放至开发板的**/boot**目录下，`reboot`重启开发板；

8. **至此我们的设备树文件已经编译完成**；

9. 重启之后，可以在开发板的`/sys/firmware/devicetree/base/`目录下看到读出的我们的设备节点；

    ```sh
    [root@imx6ull:~]# ls /sys/firmware/devicetree/base/
    #address-cells                 chosen                         gpio-keys                      mdxz_led@1                     pxp_v4l2                       soc
    #size-cells                    clocks                         interrupt-controller@00a01000  memory                         regulators                     sound
    aliases                        compatible                     leds                           model                          reserved-memory                spi4
    backlight                      cpus                           mdxz_led@0                     name                           sii902x-reset
    
    ```

10. 我们进入其他一个目录，可以看到我们在设备树文件中写入的数据；

    ```sh
    [root@imx6ull:/sys/firmware/devicetree/base/mdxz_led@0]# cat compatible
    mdxz,leddrv
    [root@imx6ull:/sys/firmware/devicetree/base/mdxz_led@0]# cat name
    mdxz_led
    [root@imx6ull:/sys/firmware/devicetree/base/mdxz_led@0]# hexdump pin
    0000000 0300 0100
    0000004
    ```

11. 进入目录`/sys/devices/soc0`，可以看到我们内核从我们的设备树提取出的设备；

    ```
    [root@imx6ull:/sys/devices/soc0]# ls
    backlight      gpio-keys      mdxz_led@0     power          regulators     sii902x-reset  soc_id         spi4           uevent
    family         machine        mdxz_led@1     pxp_v4l2       revision       soc            sound          subsystem
    
    ```

12. 我们进入其中一个目录，查看它的目录结构，注意这个的文件，待会儿我们添加完对应的驱动后，可以看到匹配成功后，这里目录会有变动；

    ```
    [root@imx6ull:/sys/devices/soc0/mdxz_led@0]# ls
    driver_override  modalias         of_node          power            subsystem        uevent
    ```

13. 到这里，我们的设备树算是加载完成；

### 编译程序

1. 进入我们的源码目录`/home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree`；

2. 直接`make -s`编译即可(可能需要根据实际环境修改内核目录和设置交叉编译工具链)

    ```sh
    # mdxz @ ubuntu in ~/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree on git:main o [21:19:30]
    $ make -s
    
    # mdxz @ ubuntu in ~/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree on git:main x [21:20:41]
    $ ls
    imx6ull_led_driver.c   imx6ull_led_driver.mod.c  imx6ull_led_driver.o  led_drv.h   led_drv.mod.c  led_drv.o        led_resource.h  ledtest.c  mdxz_led.dts   Module.symvers
    imx6ull_led_driver.ko  imx6ull_led_driver.mod.o  led_drv.c             led_drv.ko  led_drv.mod.o  led_operation.h  ledtest         Makefile   modules.order
    ```

    

3. 将编译出的`imx6ull_led_driver.ko`、`led_drv.ko`和`ledtest`文件复制进设备；

4. 至此，我们的模块也编译完成；

### 安装模块

1. 执行如下指令，安装模块；

    ```shell
    insmod led_drv.ko
    insmod imx6ull_led_driver.ko
    ```

2. 安装完成后，我们的驱动就会和设备节点提取出的platfo_device设备匹配，在虚拟文件系统中生成对应的设备；

    ```sh
    [root@imx6ull:/mnt]# ls /dev/mdxz*
    /dev/mdxz_led0  /dev/mdxz_led1
    ```

3. 在`/sys/devices/soc0`下的设备会多出一个`driver`目录

    ```sh
    [root@imx6ull:/sys/devices/soc0/mdxz_led@0]# ls
    driver           driver_override  modalias         of_node          power            subsystem        uevent
    
    ```

### 测试led

1. 在`ledtest`程序存放目录下，执行

    ```
    ./ledtest /dev/mdxz_led0 on
    ./ledtest /dev/mdxz_led0 off
    ```

2. 可以通过dmesg命令看到我们通过内核打印出的调试信息

    ```sh
    [  586.337633] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_init 116
    [  868.537599] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_open 98
    [  868.537626] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/imx6ull_led_driver.c chip_imx6_led_init line 35,
    [  868.537634] init Num of led 0
    [  868.537682] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_write 86
    [  868.537697] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/imx6ull_led_driver.c chip_imx6_led_ctl line 65, on
    [  868.537706] ctrl Num of led 0,The status is 1
    [  868.537740] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_close line 107
    
    [  875.173936] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_write 86
    [  875.173952] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/imx6ull_led_driver.c chip_imx6_led_ctl line 65, off
    [  875.173960] ctrl Num of led 0,The status is 0
    [  875.173993] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/2_GPIO/2_7_led_driver_imx6ull_device_tree/led_drv.c led_drv_close line 107
    ```

3. 至此，我们的实验结束。证明我们的设备树、驱动程序都可以正常工作。

