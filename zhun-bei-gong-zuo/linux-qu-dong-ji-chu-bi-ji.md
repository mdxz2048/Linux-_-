# Linux驱动基础笔记

## 嵌入式Linux驱动开发基础知识

## hello驱动

### 准备部分

挂载ubuntu主机的NFS目录到开发板

1.  执行如下指令，检查Ubuntu开发板的导出目录

    ```
     showmount -e
    ```

    ```
    mdxz@ubuntu:~/lzp/Embeded/NFS_DIR$ showmount -e
    Export list for ubuntu:
    /home/mdxz/lzp/Embeded/NFS_DIR 192.168.0.0/24
    ```
2.  在开发板执行如下指令，将nfs目录挂载到本地的/mnt下；

    ```
    mount -t nfs -o nolock,vers=3 192.168.0.111:/home/mdxz/lzp/Embeded/NFS_DIR /mnt
    ```
3. 其他

### 驱动开发步骤

1. 确定主设备号；
2. 定义自己的file\_operations结构体；
3. 实现对应的open/read/write等函数，填入file\_operations结构体；
4. 把file\_operations结构体告诉内核，注册驱动程序；
5. 入口函数：安装驱动时，就会去调用入口函数；
6. 出口函数：卸载驱动时，就会去调用出口函数；
7. 其他完善，提供设备信息，自动创建设备节点。
