# 2-imx6ull按键模板实验

## 实验演示

1. 在[这里](https://github.com/mdxz2048/Linux-driver-development-basics-code/tree/main/3_KEY/3_2_key_temp)下载代码;

2. 修改`MAKEFILE`中内核文件目录为实验环境的内核目录；

    ```sh
    KERN_DIR = /home/mdxz/lzp/Embeded/100ask_imx6ull-sdk/Linux-4.9.88
    ```

    

3. 编译、复制驱动模块和测试程序到NFS文件目录;

    ```shell
    make -s
    cp *.ko keytest ~/lzp/Embeded/NFS_DIR
    ```

    

4. ssh登陆到**100ask_imx6ull**开发板，挂载开发主机的NFS目录;

    ```sh
    mount -t nfs -o nolock,vers=3 192.168.0.111:/home/mdxz/lzp/Embeded/NFS_DIR /mnt
    ```

    

5. `insmod`安装驱动;

    ```
    insmod key_drv.ko
    insmod key_drv.ko
    ```

    

6. 可以通过`dmesg`查看内核打印信息;

    ```sh
    [  125.317717] key_drv: loading out-of-tree module taints kernel.
    [  125.332828] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/key_drv.c key_drv_init 86
    [  197.719185] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/key_drv.c key_drv_open 76
    [  197.730594] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/imx6ull_key_driver.c imx6ull_gpio_init line 25,
    [  197.747580] init key 0 ...
    [  197.750517] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/key_drv.c key_drv_read 69
    [  200.907858] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/key_drv.c key_drv_open 76
    [  200.919213] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/imx6ull_key_driver.c imx6ull_gpio_init line 25,
    [  200.933211] init key 1 ...
    [  200.938912] /home/mdxz/lzp/Embeded/study/Linux-driver-development-basics-code/3_KEY/3_2_key_temp/key_drv.c key_drv_read 69
    ```

    

7. 安装完成后，可以通过`ls /dev/mdxz*`来查看设备；

    ```sh
    [root@imx6ull:/mnt]# ls /dev/mdxz*
    /dev/mdxz_key0  /dev/mdxz_key1
    ```

    

8. 这时候我们的驱动中也没有动作，我们这里先演示一下；

    ```sh
    [root@imx6ull:/mnt]# ./keytest /dev/mdxz_key0
    get button : 0
    [root@imx6ull:/mnt]# ./keytest /dev/mdxz_key1
    get button : 0
    ```

    

9. 到这里，我们的`demo`程序就演示完成，这里还是主要基于LED驱动进行一个改造，下一节我们将结合GPIO驱动相关知识来实现板卡上的按键驱动。
