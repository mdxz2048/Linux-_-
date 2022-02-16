# 主机搭建NFS服务器

##

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
