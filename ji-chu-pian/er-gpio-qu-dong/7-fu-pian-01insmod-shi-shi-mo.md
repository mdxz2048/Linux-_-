# 7-副篇01：insmod是什么？

# insmod是什么

insmod是install module的缩写，是Linux一个常用命令。

我们使用的100ask_imx6ull的系统使用BusyBox进行编译，我们开发板的insmod的源码可以这里[查看](https://github.com/brgl/busybox/blob/master/modutils/insmod.c)。

> BusyBox 是一个开源（GPL）项目，提供近 400 个常用命令的简单实现，包括 `ls`、`mv`、`ln`、`mkdir`、`more`、`ps`、`gzip`、`bzip2`、`tar` 和 `grep`。它还包含了编程语言 `awk`、流编辑器 `sed`、文件系统检查工具 `fsck`、`rpm` 和 `dpkg` 软件包管理器，当然还有一个可以方便的访问所有这些命令的 shell（`sh`）。简而言之，它包含了所有 POSIX 系统需要的基本命令，以执行常见的系统维护任务以及许多用户和管理任务。

# insmod的作用

insmod用来载入模块，通过模式的方式在需要时载入内核，可使**内核精简**，**高效**，同时方便驱动程序的调试。此类载入的模块，通常为设备驱动程序。我们这里调试驱动时，多数采用了这种方式测试。

**静态加载**就是把驱动程序直接编译进内核，系统启动后可以直接调用。静态加载的缺点是调试起来比较麻烦，每次修改一个地方都要重新编译和下载内核，效率较低。若采用静态加载的驱动较多，会导致内核容量很大，浪费存储空间。
**动态加载**利用了Linux的module特性，可以在系统启动后用insmod命令添加模块（.ko），在不需要的时候用rmmod命令卸载模块，采用这种动态加载的方式便于驱动程序的调试，同时可以针对产品的功能需求，进行内核的裁剪，将不需要的驱动去除，大大减小了内核的存储容量。

# insmod的工作原理

## 内核和insmod的关系

## insmode的工作机制

### 调用关系

```c
insmod hello.ko
[Busybox_1.30.0/modutils/insmod.c]
	=>int insmod_main(int argc UNUSED_PARAM, char **argv);
        	=>bb_init_module(filename, parse_cmdline_module_options(argv, /*quote_spaces:*/ 0));
[Busybox_1.30.0/modutils/modutils.c]
	=>int FAST_FUNC bb_init_module(const char *filename, const char *options);
		=>open(filename, O_RDONLY | O_CLOEXEC);
		=>finit_module(fd, options, 0) != 0;
		=># define finit_module(fd, uargs, flags) syscall(__NR_finit_module, fd, uargs, flags)
到这里，可以看到，finit_module调用syscall进行了系统调用,后边我们就转到内核源码,找到__NR_finit_module对应的内核调用编号；
[Linux-4.9.88/arch/arm/include/uapi/asm/unistd.h]
	=>#define __NR_init_module		(__NR_SYSCALL_BASE+128)
[Linux-4.9.88/arch/arm/kernel/calls.S]
	=>CALL(sys_init_module)
[Linux-4.9.88/include/linux/syscalls.h]	
	=>asmlinkage long sys_init_module(void __user *umod, unsigned long len,const char __user *uargs);	
	=>#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
	=> #define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
[/Linux-4.9.88/kernel/module.c]
	=>SYSCALL_DEFINE3(finit_module, int, fd, const char __user *, uargs, int, flags);
		=>kernel_read_file_from_fd(fd, &hdr, &size, INT_MAX,READING_MODULE);
			=>load_module(&info, uargs, flags);
				=>static int load_module(struct load_info *info, const char __user *uargs,int flags)
					=>module_sig_check(info, flags);
					=>elf_header_check(info);
					=>layout_and_allocate(info, flags);//这里会将内存的二进制代码格式化到module结构体并返回；
					=>setup_modinfo(mod, info);
					=>simplify_symbols(mod, info);
					=>do_init_module(mod);
						=>static noinline int do_init_module(struct module *mod)
							=>do_one_initcall(mod->init);
								=>fn();//这个就是实际的初始化调用的地方
```



### 分析

1. 加载内核insmod hello.ko

2. insmod命令是由busybox编译来的，我们在**Busybox_1.30.0**中找到相关代码；

3. insmod_main()位于*[Busybox_1.30.0/modutils/insmod.c]*中：

   - 这里传入参数为 `hello.ko`
   - 调用`bb_init_module()`来初始化模块；

   ```c
   int insmod_main(int argc UNUSED_PARAM, char **argv)
   {
   	char *filename;
   	int rc;
   
   	/* Compat note:
   	 * 2.6 style insmod has no options and required filename
   	 * (not module name - .ko can't be omitted).
   	 * 2.4 style insmod can take module name without .o
   	 * and performs module search in default directories
   	 * or in $MODPATH.
   	 */
   
   	IF_FEATURE_2_4_MODULES(
   		getopt32(argv, INSMOD_OPTS INSMOD_ARGS);
   		argv += optind - 1;
   	);
   
   	filename = *++argv;
   	if (!filename)
   		bb_show_usage();
   	/*初始化模块*/
   	rc = bb_init_module(filename, parse_cmdline_module_options(argv, /*quote_spaces:*/ 0));
   	if (rc)
   		bb_error_msg("can't insert '%s': %s", filename, moderror(rc));
   
   	return rc;
   }
   ```

   

4. `bb_init_module（）`位于*[Busybox_1.30.0/modutils/modutils.c]*中，

   - 我们在[*Linux-4.9.88/arch/alpha/include/uapi/asm/unistd.h*](https://elixir.bootlin.com/linux/v4.9.88/source/include/uapi/asm-generic/unistd.h)中可以看到`__NR_finit_module`被定义；

   - 因此，这里调用`open(filename, O_RDONLY | O_CLOEXEC)`打开模块；

   - 通过`finit_module(fd, options, 0)`接口来初始化模块；

   - 在当前文件下，可以看到文件刚开始有如下宏定义，在这里insmod通过[syscall](http://gityuan.com/2016/05/21/syscall/)系统调用陷入从用户空间陷入内核空间；

     ```
     #define finit_module(mod, len, opts) syscall(__NR_init_module, mod, len, opts)
     ```

   ```c
   /* Return:
    * 0 on success,
    * -errno on open/read error,
    * errno on init_module() error
    */
   int FAST_FUNC bb_init_module(const char *filename, const char *options)
   {
   	size_t image_size;
   	char *image;
   	int rc;
   	bool mmaped;
   
   	if (!options)
   		options = "";
   
   //TODO: audit bb_init_module_24 to match error code convention
   #if ENABLE_FEATURE_2_4_MODULES
   	if (get_linux_version_code() < KERNEL_VERSION(2,6,0))
   		return bb_init_module_24(filename, options);
   #endif
   
   	/*
   	 * First we try finit_module if available.  Some kernels are configured
   	 * to only allow loading of modules off of secure storage (like a read-
   	 * only rootfs) which needs the finit_module call.  If it fails, we fall
   	 * back to normal module loading to support compressed modules.
   	 */
       /*Linux-4.9.88/arch/alpha/include/uapi/asm/unistd.h中定义__NR_finit_module*/
   # ifdef __NR_finit_module
   	{
   		int fd = open(filename, O_RDONLY | O_CLOEXEC);
   		if (fd >= 0) {
   			rc = finit_module(fd, options, 0) != 0;
   			close(fd);
   			if (rc == 0)
   				return rc;
   		}
   	}
   # endif
   
   	image_size = INT_MAX - 4095;
   	mmaped = 0;
   	image = try_to_mmap_module(filename, &image_size);
   	if (image) {
   		mmaped = 1;
   	} else {
   		errno = ENOMEM; /* may be changed by e.g. open errors below */
   		image = xmalloc_open_zipped_read_close(filename, &image_size);
   		if (!image)
   			return -errno;
   	}
   
   	errno = 0;
   	init_module(image, image_size, options);
   	rc = errno;
   	if (mmaped)
   		munmap(image, image_size);
   	else
   		free(image);
   	return rc;
   }
   ```

5. 接下来，我们打开我们编译好的内核来追踪，这里内核版本是**Linux-4.9.88**。这里我们做的工作无非是关注一个系统调用如何在内核中工作的，不妨参考一下[这篇](https://www.linuxbnb.net/home/adding-a-system-call-to-linux-arm-architecture/)博客,这里描述了如何向Linux内核添加一个系统调用。

6. 看完之后，我们继续分析：

7. `__NR_init_module`这个宏定义在*[Linux-4.9.88/arch/arm/include/uapi/asm/unistd.h]*文件中；；

   - 这个文件是unix系统提供的头文件，用来对系统调用进行封装；
   - 可以看到这里系统调用编号是**基地址+128**,这里对应的是

   ```c
   define __NR_init_module		(__NR_SYSCALL_BASE+128)
   ```

8. 系统调用可以参考*[[Linux-4.9.88/arch/arm/kernel/calls.S]](https://elixir.bootlin.com/linux/v4.9.88/source/arch/arm/kernel/calls.S)*文件，这里可以看到我们所有的系统调用对应的接口，128对应的是`sys_init_module()`；

   ```c
   /* 125 */	CALL(sys_mprotect)
   		CALL(sys_sigprocmask)
   		CALL(sys_ni_syscall)		/* was sys_create_module */
   		CALL(sys_init_module)
   		CALL(sys_delete_module)
   ```

9. 接下来，我们在[[Linux-4.9.88/include/linux/syscalls.h]](https://elixir.bootlin.com/linux/v4.9.88/source/include/linux/syscalls.h)中，可以看到这里应用了

10. 其他





