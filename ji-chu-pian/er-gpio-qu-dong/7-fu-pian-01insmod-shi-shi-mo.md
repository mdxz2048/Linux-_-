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
	=>SYSCALL_DEFINE3(init_module, void __user *, umod,unsigned long, len, const char __user *, uargs)
		=>copy_module_from_user(umod, len, &info);
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

9. 接下来，我们在[[Linux-4.9.88/include/linux/syscalls.h]](https://elixir.bootlin.com/linux/v4.9.88/source/include/linux/syscalls.h)中，可以看到有对sys_init_module的定义，sys_init_module的功能是分配内核存储空间（kernel memory）给相关的模块；

   ```c
   asmlinkage long sys_init_module(void __user *umod, unsigned long len,const char __user *uargs);
   ```

   

10. `sys_init_module`系统调用在[kernel/module.c]中，这里主要工作如下：

    - 从用户空间拷贝模块镜像到内核空间：`copy_module_from_user（）`
    - 加载模块:`load_module（）`

    ```c
    SYSCALL_DEFINE3(init_module, void __user *, umod,
    		unsigned long, len, const char __user *, uargs)
    {
    	int err;
    	struct load_info info = { };
    	/*这里不会初始化模块*/
    	err = may_init_module();
    	if (err)
    		return err;
    
    	pr_debug("init_module: umod=%p, len=%lu, uargs=%p\n",
    	       umod, len, uargs);
    
    	err = copy_module_from_user(umod, len, &info);
    	if (err)
    		return err;
    
    	return load_module(&info, uargs, 0);
    }
    ```

    

11. `load_module()`函数同样位于[kernel/module.c]中，这里的任务是:

    - 分配空间给模块
    - 检查ELF文件镜像
    - 解析镜像到mod结构体，检查符号表、变量；
    - 调用`do_init_module()`初始化模组；

    ```
    /* Allocate and load the module: note that size of section 0 is always
       zero, and we rely on this for optional sections. */
    static int load_module(struct load_info *info, const char __user *uargs,
    		       int flags)
    {
    	struct module *mod;
    	long err;
    	char *after_dashes;
    	/*模组签名检查，需要内核中配置CONFIG_MODULE_SIG，我们这里没有配置，因此跳过*/
    	err = module_sig_check(info, flags);
    	if (err)
    		goto free_copy;
    	/*ELF镜像的头校验*/
    	err = elf_header_check(info);
    	if (err)
    		goto free_copy;
    	
    	/*为模块的各个section分配空间.这里会将内存的二进制代码格式化到module结构体并返回；*/
    	mod = .layout_and_allocate(info, flags);
    	if (IS_ERR(mod)) {
    		err = PTR_ERR(mod);
    		goto free_copy;
    	}
    
    	/*查是否有同名模块已加载，如果没有则将mod加入到链表中*/
    	err = add_unformed_module(mod);
    	if (err)
    		goto free_module;
    
    #ifdef CONFIG_MODULE_SIG
    	mod->sig_ok = info->sig_ok;
    	if (!mod->sig_ok) {
    		pr_notice_once("%s: module verification failed: signature "
    			       "and/or required key missing - tainting "
    			       "kernel\n", mod->name);
    		add_taint_module(mod, TAINT_UNSIGNED_MODULE, LOCKDEP_STILL_OK);
    	}
    #endif
    	
    	/* 这里是将mod变量为各个CPU存放一份。模块的per-cpu section是ELF文件中一个特殊的section，属于data区，模块加载时，会根据系统中CPU个数，将这个 section中的数据复制相应的份数，存放在CORE section区域。这个主要在SMP系统中，不同CPU可以访问模块per-cpu section中的数据而无需使用CPU间的互斥机制。 */
    	err = percpu_modalloc(mod, info);
    	if (err)
    		goto unlink_mod;
    
    	/* 初始化卸载模块相关的成员变量 */
    	err = module_unload_init(mod);
    	if (err)
    		goto unlink_mod;
    	
    	init_param_lock(mod);
    
    	/* Now we've got everything in the final locations, we can
    	 * find optional sections. */
    	err = find_module_sections(mod, info);
    	if (err)
    		goto free_unload;
    
    	err = check_module_license_and_versions(mod);
    	if (err)
    		goto free_unload;
    
    	/* Set up MODINFO_ATTR fields */
    	setup_modinfo(mod, info);
    
    	/* Fix up syms, so that st_value is a pointer to location. */
    	err = simplify_symbols(mod, info);
    	if (err < 0)
    		goto free_modinfo;
    
    	err = apply_relocations(mod, info);
    	if (err < 0)
    		goto free_modinfo;
    
    	err = post_relocation(mod, info);
    	if (err < 0)
    		goto free_modinfo;
    
    	flush_module_icache(mod);
    
    	/* Now copy in args */
    	mod->args = strndup_user(uargs, ~0UL >> 1);
    	if (IS_ERR(mod->args)) {
    		err = PTR_ERR(mod->args);
    		goto free_arch_cleanup;
    	}
    
    	dynamic_debug_setup(info->debug, info->num_debug);
    
    	/* Ftrace init must be called in the MODULE_STATE_UNFORMED state */
    	ftrace_module_init(mod);
    
    	/* Finally it's fully formed, ready to start executing. */
    	err = complete_formation(mod, info);
    	if (err)
    		goto ddebug_cleanup;
    
    	err = prepare_coming_module(mod);
    	if (err)
    		goto bug_cleanup;
    
    	/* Module is ready to execute: parsing args may do that. */
    	after_dashes = parse_args(mod->name, mod->args, mod->kp, mod->num_kp,
    				  -32768, 32767, mod,
    				  unknown_module_param_cb);
    	if (IS_ERR(after_dashes)) {
    		err = PTR_ERR(after_dashes);
    		goto coming_cleanup;
    	} else if (after_dashes) {
    		pr_warn("%s: parameters '%s' after `--' ignored\n",
    		       mod->name, after_dashes);
    	}
    
    	/* Link in to syfs. */
    	err = mod_sysfs_setup(mod, info, mod->kp, mod->num_kp);
    	if (err < 0)
    		goto coming_cleanup;
    
    	if (is_livepatch_module(mod)) {
    		err = copy_module_elf(mod, info);
    		if (err < 0)
    			goto sysfs_cleanup;
    	}
    
    	/* Get rid of temporary copy. */
    	free_copy(info);
    
    	/* Done! */
    	trace_module_load(mod);
    	/*初始化module*/
    	return do_init_module(mod);
    
     sysfs_cleanup:
    	mod_sysfs_teardown(mod);
     coming_cleanup:
    	blocking_notifier_call_chain(&module_notify_list,
    				     MODULE_STATE_GOING, mod);
    	klp_module_going(mod);
     bug_cleanup:
    	/* module_bug_cleanup needs module_mutex protection */
    	mutex_lock(&module_mutex);
    	module_bug_cleanup(mod);
    	mutex_unlock(&module_mutex);
    
    	/* we can't deallocate the module until we clear memory protection */
    	module_disable_ro(mod);
    	module_disable_nx(mod);
    
     ddebug_cleanup:
    	dynamic_debug_remove(info->debug);
    	synchronize_sched();
    	kfree(mod->args);
     free_arch_cleanup:
    	module_arch_cleanup(mod);
     free_modinfo:
    	free_modinfo(mod);
     free_unload:
    	module_unload_free(mod);
     unlink_mod:
    	mutex_lock(&module_mutex);
    	/* Unlink carefully: kallsyms could be walking list. */
    	list_del_rcu(&mod->list);
    	mod_tree_remove(mod);
    	wake_up_all(&module_wq);
    	/* Wait for RCU-sched synchronizing before releasing mod->list. */
    	synchronize_sched();
    	mutex_unlock(&module_mutex);
     free_module:
    	/*
    	 * Ftrace needs to clean up what it initialized.
    	 * This does nothing if ftrace_module_init() wasn't called,
    	 * but it must be called outside of module_mutex.
    	 */
    	ftrace_release_mod(mod);
    	/* Free lock-classes; relies on the preceding sync_rcu() */
    	lockdep_free_key_range(mod->core_layout.base, mod->core_layout.size);
    
    	module_deallocate(mod, info);
     free_copy:
    	free_copy(info);
    	return err;
    }
    ```

    

12. do_init_module()函数的定义在`[kernel/module.c]`中，这里的工作主要是：

    - `do_one_initcall(mod->init)`调用模块的初始化函数，进行初始化；

    ```c
    /*
     * This is where the real work happens.
     *
     * Keep it uninlined to provide a reliable breakpoint target, e.g. for the gdb
     * helper command 'lx-symbols'.
     */
    static noinline int do_init_module(struct module *mod)
    {
    	int ret = 0;
    	struct mod_initfree *freeinit;
    
    	freeinit = kmalloc(sizeof(*freeinit), GFP_KERNEL);
    	if (!freeinit) {
    		ret = -ENOMEM;
    		goto fail;
    	}
    	freeinit->module_init = mod->init_layout.base;
    
    	/*
    	 * We want to find out whether @mod uses async during init.  Clear
    	 * PF_USED_ASYNC.  async_schedule*() will set it.
    	 */
    	current->flags &= ~PF_USED_ASYNC;
    
    	do_mod_ctors(mod);
    	/* Start the module */
    	if (mod->init != NULL)
    		ret = do_one_initcall(mod->init);
    	if (ret < 0) {
    		goto fail_free_freeinit;
    	}
    	if (ret > 0) {
    		pr_warn("%s: '%s'->init suspiciously returned %d, it should "
    			"follow 0/-E convention\n"
    			"%s: loading module anyway...\n",
    			__func__, mod->name, ret, __func__);
    		dump_stack();
    	}
    
    	/* Now it's a first class citizen! */
    	mod->state = MODULE_STATE_LIVE;
    	blocking_notifier_call_chain(&module_notify_list,
    				     MODULE_STATE_LIVE, mod);
    
    	/*
    	 * We need to finish all async code before the module init sequence
    	 * is done.  This has potential to deadlock.  For example, a newly
    	 * detected block device can trigger request_module() of the
    	 * default iosched from async probing task.  Once userland helper
    	 * reaches here, async_synchronize_full() will wait on the async
    	 * task waiting on request_module() and deadlock.
    	 *
    	 * This deadlock is avoided by perfomring async_synchronize_full()
    	 * iff module init queued any async jobs.  This isn't a full
    	 * solution as it will deadlock the same if module loading from
    	 * async jobs nests more than once; however, due to the various
    	 * constraints, this hack seems to be the best option for now.
    	 * Please refer to the following thread for details.
    	 *
    	 * http://thread.gmane.org/gmane.linux.kernel/1420814
    	 */
    	if (!mod->async_probe_requested && (current->flags & PF_USED_ASYNC))
    		async_synchronize_full();
    
    	mutex_lock(&module_mutex);
    	/* Drop initial reference. */
    	module_put(mod);
    	trim_init_extable(mod);
    #ifdef CONFIG_KALLSYMS
    	/* Switch to core kallsyms now init is done: kallsyms may be walking! */
    	rcu_assign_pointer(mod->kallsyms, &mod->core_kallsyms);
    #endif
    	module_enable_ro(mod, true);
    	mod_tree_remove_init(mod);
    	disable_ro_nx(&mod->init_layout);
    	module_arch_freeing_init(mod);
    	mod->init_layout.base = NULL;
    	mod->init_layout.size = 0;
    	mod->init_layout.ro_size = 0;
    	mod->init_layout.ro_after_init_size = 0;
    	mod->init_layout.text_size = 0;
    	/*
    	 * We want to free module_init, but be aware that kallsyms may be
    	 * walking this with preempt disabled.  In all the failure paths, we
    	 * call synchronize_sched(), but we don't want to slow down the success
    	 * path, so use actual RCU here.
    	 */
    	call_rcu_sched(&freeinit->rcu, do_free_init);
    	mutex_unlock(&module_mutex);
    	wake_up_all(&module_wq);
    
    	return 0;
    
    fail_free_freeinit:
    	kfree(freeinit);
    fail:
    	/* Try to protect us from buggy refcounters. */
    	mod->state = MODULE_STATE_GOING;
    	synchronize_sched();
    	module_put(mod);
    	blocking_notifier_call_chain(&module_notify_list,
    				     MODULE_STATE_GOING, mod);
    	klp_module_going(mod);
    	ftrace_release_mod(mod);
    	free_module(mod);
    	wake_up_all(&module_wq);
    	return ret;
    }
    ```

    

13. do_one_initcall()函数在[Linux-4.9.88/init/main.c]中，这个函数比较简单，主要就是调用传入参数`initcall_t fn`，即真正实现mod中的初始化函数。

    ```c
    int __init_or_module do_one_initcall(initcall_t fn)
    {
    	int count = preempt_count();
    	int ret;
    	char msgbuf[64];
    
    	if (initcall_blacklisted(fn))
    		return -EPERM;
    
    	if (initcall_debug)
    		ret = do_one_initcall_debug(fn);
    	else
    		ret = fn();
    
    	msgbuf[0] = 0;
    
    	if (preempt_count() != count) {
    		sprintf(msgbuf, "preemption imbalance ");
    		preempt_count_set(count);
    	}
    	if (irqs_disabled()) {
    		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
    		local_irq_enable();
    	}
    	WARN(msgbuf[0], "initcall %pF returned with %s\n", fn, msgbuf);
    
    	add_latent_entropy();
    	return ret;
    }
    ```

    

14. 其他





