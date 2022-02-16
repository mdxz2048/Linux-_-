# 2-hello驱动

1.  确定主设备号

    ```
     static int major = 0;
    ```
2.  定义自己的file\_operations结构体；

    ```
     static struct file_operations hello_drv = {     .owner   = THIS_MODULE,     .open    = hello_drv_open,     .read    = hello_drv_read,     .write   = hello_drv_write,     .release = hello_drv_close, };
    ```
3.  实现对应的open/read/write等函数，填入file\_operations结构体；

    ```
     static ssize_t hello_drv_read (struct file *file, char __user *buf, size_t size, loff_t *offset) {     int err;     printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     err = copy_to_user(buf, kernel_buf, MIN(1024, size));     return MIN(1024, size); } ​ static ssize_t hello_drv_write (struct file *file, const char __user *buf, size_t size, loff_t *offset) {     int err;     printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     err = copy_from_user(kernel_buf, buf, MIN(1024, size));     return MIN(1024, size); } ​ static int hello_drv_open (struct inode *node, struct file *file) {     printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     return 0; } ​ static int hello_drv_close (struct inode *node, struct file *file) {     printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     return 0; }
    ```
4. 把file\_operations结构体告诉内核，注册驱动程序；
5.  入口函数：安装驱动时，就会去调用入口函数；

    ```
     static int __init hello_init(void) {     int err;          printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     major = register_chrdev(0, "hello", &hello_drv);  /* /dev/hello */ ​ ​     hello_class = class_create(THIS_MODULE, "hello_class");     err = PTR_ERR(hello_class);     if (IS_ERR(hello_class)) {         printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);         unregister_chrdev(major, "hello");         return -1;     }          device_create(hello_class, NULL, MKDEV(major, 0), NULL, "hello"); /* /dev/hello */          return 0; }
    ```
6.  出口函数：卸载驱动时，就会去调用出口函数；

    ```
     static void __exit hello_exit(void) {     printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);     device_destroy(hello_class, MKDEV(major, 0));     class_destroy(hello_class);     unregister_chrdev(major, "hello"); } ​
    ```
7.  其他完善，提供设备信息，自动创建设备节点。

    ```
     module_exit(hello_exit); ​ MODULE_LICENSE("GPL");
    ```

## 编写测试程序-hello\_drv\_test.c

测试程序用来测试驱动是否可以正常工作。

1.  判断参数；

    ```
         if (argc < 2)      {         printf("Usage: %s -w <string>\n", argv[0]);         printf("       %s -r\n", argv[0]);         return -1;     } ​
    ```
2.  打开设备；

    ```
         fd = open("/dev/hello", O_RDWR);     if (fd == -1)     {         printf("can not open file /dev/hello\n");         return -1;     } ​
    ```
3.  操作设备；

    ```
         if ((0 == strcmp(argv[1], "-w")) && (argc == 3))     {         len = strlen(argv[2]) + 1;         len = len < 1024 ? len : 1024;         write(fd, argv[2], len);     }     else     {         len = read(fd, buf, 1024);               buf[1023] = '\0';         printf("APP read : %s\n", buf);     }      ​
    ```
4.  关闭设备

    ```
         close(fd);
    ```
5. 其他
