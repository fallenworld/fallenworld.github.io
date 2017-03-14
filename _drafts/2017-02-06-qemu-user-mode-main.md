---
layout: post
title: qemu user mode main函数源码分析
categories: qemu
tags: qemu
---

* content
{:toc}

## qemu两种模式 ##

qemu是什么：[https://zh.wikipedia.org/wiki/QEMU](https://zh.wikipedia.org/wiki/QEMU "维基百科qemu")

简单来讲qemu就是一个模拟器，可以在一种架构上模拟另外一种架构，例如可以在arm上模拟x86。其核心的实现方式是使用动态二进制翻译（二进制翻译：把一种架构的机器码翻译成另外一种架构的机器码）

qemu在使用的时候有两种模式：

- User mode用户模式：直接用qemu运行Linux的二进制文件，不需要模拟整个计算机，只要模拟个CPU，然后直接加载执行Linux的elf文件就行了。
- System mode系统模式：这个模式下需要模拟所有的计算机硬件，包括CPU、网卡、硬盘等，这时候执行一个程序要从操作系统引导开始，这里相当于模拟了一台完整的虚拟计算机。

## user mode 下的入口 ##

qemu中和Linux user mode相关的代码在源码根目录下的linux-user/中

其中入口的main函数在linux-user/main.c里

## main函数分析 ##

这里假设我们要模拟的taget架构为i386(x86)

整个main函数的流程如下：

### 1. 初始化QOM、TRACE模块，初始化cpu列表

```c
    module_call_init(MODULE_INIT_TRACE);
    qemu_init_cpu_list();
    module_call_init(MODULE_INIT_QOM);
```
模块：http://www.voidcn.com/blog/fenglinyixuan/article/p-6024461.html
http://blog.csdn.net/u011364612/article/details/53581501

qemu_init_cpu_list()初始化了一些锁和信号量，该函数定义如下：

```c
void qemu_init_cpu_list(void)
{
    /* This is needed because qemu_init_cpu_list is also called by the
     * child process in a fork.  */
    pending_cpus = 0;
    qemu_mutex_init(&qemu_cpu_list_lock);
    qemu_cond_init(&exclusive_cond);
    qemu_cond_init(&exclusive_resume);
    qemu_cond_init(&qemu_work_cond);
}
```

这些锁和信号量用于互斥访问CPUTailQ cpus这个变量，cpus是一个元素为CPUState的链表。

综上，cpus是一个cpu列表，qemu_init_cpu_list()初始化了它

### 2. 创建环境变量链表
```c
    if ((envlist = envlist_create()) == NULL) {              //创建环境变量链表
        (void) fprintf(stderr, "Unable to allocate envlist\n");
        exit(EXIT_FAILURE);
    }
    /* add current environment into the list */
    for (wrk = (*_NSGetEnviron()); *wrk != NULL; wrk++) {    //环境变量复制到链表中
        (void) envlist_setenv(envlist, *wrk);
    }
```
envlist的类型如下：

```c
struct envlist {
	envlist_entry* el_entries;	/* actual entries */
	size_t el_count; 			/* number of entries */
};

struct envlist_entry {
	const char *ev_var; 	/* actual env value */
    struct envlist_entry *le_next;   /* next element */                     
    struct envlist_entry **le_prev;  /* address of previous next element */
};
```

可见envlist是一个环境变量的链表

envlist_create()创建链表，for循环把当前进程的环境变量放入envlist中

### 3. 获取host平台栈限长

```c
    /* Read the stack limit from the kernel.  If it's "unlimited",
       then we can do little else besides use the default.  */
    {
        struct rlimit lim;
        if (getrlimit(RLIMIT_STACK, &lim) == 0
            && lim.rlim_cur != RLIM_INFINITY
            && lim.rlim_cur == (target_long)lim.rlim_cur) {
            guest_stack_size = lim.rlim_cur;
        }
    }
```
getrlimit函数可参考：[http://man7.org/linux/man-pages/man2/getrlimit.2.html](http://man7.org/linux/man-pages/man2/getrlimit.2.html)

### 4. 初始化变量和全局设置

```c
    cpu_model = NULL;
    srand(time(NULL));
    qemu_add_opts(&qemu_trace_opts); 	//将一些调试用的选项加入配置中
    optind = parse_args(argc, argv); 	//分析qemu参数
    if (!trace_init_backends()) {        //初始化调试输出
        exit(1);
    }
    trace_init_file(trace_file); 		//初始化调试输出文件
    /* Zero out regs */
    memset(regs, 0, sizeof(struct target_pt_regs));
    /* Zero out image_info */
    memset(info, 0, sizeof(struct image_info));
    memset(&bprm, 0, sizeof (bprm));
    /* Scan interp_prefix dir for replacement files. */
    init_paths(interp_prefix);           //设置解释器的路径
    init_qemu_uname_release();           //初始化系统信息
```

qemu_add_opts(&qemu_trace_opts)是将一些调试输出用的选项加入全局配置中

qemu_trace_opts的定义如下：

```c
QemuOptsList qemu_trace_opts = {
    .name = "trace",
    .implied_opt_name = "enable",
    .head = QTAILQ_HEAD_INITIALIZER(qemu_trace_opts.head),
    .desc = {
        {
            .name = "enable",
            .type = QEMU_OPT_STRING,
        },
        {
            .name = "events",
            .type = QEMU_OPT_STRING,
        },{
            .name = "file",
            .type = QEMU_OPT_STRING,
        },
        { /* end of list */ }
    },
};
```
可以看到QemuOptsList这个结构是一些qemu的一些选项设置，qemu_trace_opts就是和trace(调试输出)相关的选项设置
optind = parse_args(argc, argv)是分析为qemu传入的命令行参数，根据传入的参数得到一个序号存到optind中，序号和对应的参数参考下表：

![](http://blog.chinaunix.net/attachment/201112/24/20940095_13247362205JtQ.png)
（表来源：http://blog.csdn.net/ustc_dylan/article/details/7100439）

init_paths(interp_prefix)设置elf文件解释器的路径

init_qemu_uname_release()会调用Linux中的uname函数来获取一些系统信息

（uname函数参考：[http://man7.org/linux/man-pages/man2/uname.2.html](http://man7.org/linux/man-pages/man2/uname.2.html)）

### 5. 初始化CPU

```c
    if (cpu_model == NULL) {
        cpu_model = "qemu32";
    }
    tcg_exec_init(0);            //初始化tcg动态翻译器
    /* NOTE: we need to init the CPU at this stage to get
       qemu_host_page_size */
    cpu = cpu_init(cpu_model);   //初始化i386 CPU
    if (!cpu) {
        fprintf(stderr, "Unable to find CPU definition\n");
        exit(EXIT_FAILURE);
    }
    env = cpu->env_ptr;
    cpu_reset(cpu);              //CPU复位
    thread_cpu = cpu;
```

### 6. 操作环境变量

```c
    if (getenv("QEMU_STRACE")) {
        do_strace = 1;
    }
    if (getenv("QEMU_RAND_SEED")) {
        handle_arg_randseed(getenv("QEMU_RAND_SEED"));
    }
    target_environ = envlist_to_environ(envlist, NULL);
    envlist_free(envlist);
```

两个if语句根据环境变量设置了一些变量的值，接着环境变量链表被转换为了char**类型

### 7. 设置target平台内存空间

```c
    /*
     * Now that page sizes are configured in cpu_init() we can do
     * proper page alignment for guest_base.
     */
    guest_base = HOST_PAGE_ALIGN(guest_base);
    if (reserved_va || have_guest_base) {
        guest_base = init_guest_space(guest_base, reserved_va, 0,
                                      have_guest_base);
        if (guest_base == (unsigned long)-1) {
            fprintf(stderr, "Unable to reserve 0x%lx bytes of virtual address "
                    "space for use as guest address space (check your virtual "
                    "memory ulimit setting or reserve less using -R option)\n",
                    reserved_va);
            exit(EXIT_FAILURE);
        }
        if (reserved_va) {
            mmap_next_start = reserved_va;
        }
    }
```

HOST_PAGE_ALIGN宏的定义如下：

```c
#define HOST_PAGE_ALIGN(addr) (((addr) + qemu_host_page_size - 1) & qemu_host_page_mask)
```

这个宏是向下取某一地址的页起始地址，也就是把某个地址数值向下对齐到页边界

因此guest_base = HOST_PAGE_ALIGN(guest_base)这一句是将guest_base对齐到host平台的页边界

init_guest_space是为target平台在host平台的虚拟内存空间中分配reserved_va大小的空间


### 8. 获取host平台内存映射最小地址

```c
    /*
     * Read in mmap_min_addr kernel parameter.  This value is used
     * When loading the ELF image to determine whether guest_base
     * is needed.  It is also used in mmap_find_vma.
     */
    {
        FILE *fp;

        if ((fp = fopen("/proc/sys/vm/mmap_min_addr", "r")) != NULL) {
            unsigned long tmp;
            if (fscanf(fp, "%lu", &tmp) == 1) {
                mmap_min_addr = tmp;
                qemu_log_mask(CPU_LOG_PAGE, "host mmap_min_addr=0x%lx\n", mmap_min_addr);
            }
            fclose(fp);
        }
    }
```

通过读mmap_min_addr这个文件，可获取到host平台中进行内存映射时可用的最小地址

### 9. 设置target平台命令行参数

```c
    /*
     * Prepare copy of argv vector for target.
     */
    target_argc = argc - optind;
    target_argv = calloc(target_argc + 1, sizeof (char *));
    if (target_argv == NULL) {
	(void) fprintf(stderr, "Unable to allocate memory for target_argv\n");
	exit(EXIT_FAILURE);
    }
    /*
     * If argv0 is specified (using '-0' switch) we replace
     * argv[0] pointer with the given one.
     */
    i = 0;
    if (argv0 != NULL) {
        target_argv[i++] = strdup(argv0);
    }
    for (; i < target_argc; i++) {
        target_argv[i] = strdup(argv[optind + i]);
    target_argv[target_argc] = NULL;
```

### 10. 初始化任务状态

```c
    ts = g_new0(TaskState, 1);
    init_task_state(ts);
    /* build Task State */
    ts->info = info;    //设置任务的关联的文件映像信息
    ts->bprm = &bprm;   //设置二进制文件的参数
    cpu->opaque = ts;
    task_settid(ts);    //获取一个任务id
```

TaskState是保存任务状态的结构体，类似Linux内核中的task_struct结构体

### 11. 加载ELF文件

```c
    execfd = qemu_getauxval(AT_EXECFD);
    if (execfd == 0) {
        execfd = open(filename, O_RDONLY);
        if (execfd < 0) {
            printf("Error while loading %s: %s\n", filename, strerror(errno));
            _exit(EXIT_FAILURE);
        }
    }

    ret = loader_exec(execfd, filename, target_argv, target_environ, regs,
        info, &bprm);
    if (ret != 0) {
        printf("Error while loading %s: %s\n", filename, strerror(-ret));
        _exit(EXIT_FAILURE);
    }

    for (wrk = target_environ; *wrk; wrk++) {
        free(*wrk);
    }

    free(target_environ);

    if (qemu_loglevel_mask(CPU_LOG_PAGE)) {
        qemu_log("guest_base  0x%lx\n", guest_base);
        log_page_dump();

        qemu_log("start_brk   0x" TARGET_ABI_FMT_lx "\n", info->start_brk);
        qemu_log("end_code    0x" TARGET_ABI_FMT_lx "\n", info->end_code);
        qemu_log("start_code  0x" TARGET_ABI_FMT_lx "\n",
                 info->start_code);
        qemu_log("start_data  0x" TARGET_ABI_FMT_lx "\n",
                 info->start_data);
        qemu_log("end_data    0x" TARGET_ABI_FMT_lx "\n", info->end_data);
        qemu_log("start_stack 0x" TARGET_ABI_FMT_lx "\n",
                 info->start_stack);
        qemu_log("brk         0x" TARGET_ABI_FMT_lx "\n", info->brk);
        qemu_log("entry       0x" TARGET_ABI_FMT_lx "\n", info->entry);
    }
```

核心的进行加载的语句是：

```c
loader_exec(execfd, filename, target_argv, target_environ, regs,
        info, &bprm);
```

关于loader_exec函数的详细分析可见：

加载完成后输出了一些关于所加载文件的信息的log

### 12. 初始化信号、初始化系统调用
```c
    target_set_brk(info->brk);
    syscall_init();
    signal_init();
```

### 13. 


### 14. 初始化寄存器、描述符、描述符表
```c
    env->cr[0] = CR0_PG_MASK | CR0_WP_MASK | CR0_PE_MASK;
    env->hflags |= HF_PE_MASK | HF_CPL_MASK;
    if (env->features[FEAT_1_EDX] & CPUID_SSE) {
        env->cr[4] |= CR4_OSFXSR_MASK;
        env->hflags |= HF_OSFXSR_MASK;
    }
    /* flags setup : we activate the IRQs by default as in user mode */
    env->eflags |= IF_MASK;

    /* linux register setup */
    env->regs[R_EAX] = regs->eax;
    env->regs[R_EBX] = regs->ebx;
    env->regs[R_ECX] = regs->ecx;
    env->regs[R_EDX] = regs->edx;
    env->regs[R_ESI] = regs->esi;
    env->regs[R_EDI] = regs->edi;
    env->regs[R_EBP] = regs->ebp;
    env->regs[R_ESP] = regs->esp;
    env->eip = regs->eip;
	
    /* linux interrupt setup */
    env->idt.limit = 255;
    env->idt.base = target_mmap(0, sizeof(uint64_t) * (env->idt.limit + 1),
                                PROT_READ|PROT_WRITE,
                                MAP_ANONYMOUS|MAP_PRIVATE, -1, 0);
    idt_table = g2h(env->idt.base);
    set_idt(0, 0);
    set_idt(1, 0);
    set_idt(2, 0);
    set_idt(3, 3);
    set_idt(4, 3);
    set_idt(5, 0);
    set_idt(6, 0);
    set_idt(7, 0);
    set_idt(8, 0);
    set_idt(9, 0);
    set_idt(10, 0);
    set_idt(11, 0);
    set_idt(12, 0);
    set_idt(13, 0);
    set_idt(14, 0);
    set_idt(15, 0);
    set_idt(16, 0);
    set_idt(17, 0);
    set_idt(18, 0);
    set_idt(19, 0);
    set_idt(0x80, 3);

    /* linux segment setup */
    {
        uint64_t *gdt_table;
        env->gdt.base = target_mmap(0, sizeof(uint64_t) * TARGET_GDT_ENTRIES,
                                    PROT_READ|PROT_WRITE,
                                    MAP_ANONYMOUS|MAP_PRIVATE, -1, 0);
        env->gdt.limit = sizeof(uint64_t) * TARGET_GDT_ENTRIES - 1;
        gdt_table = g2h(env->gdt.base);
        write_dt(&gdt_table[__USER_CS >> 3], 0, 0xfffff,
                 DESC_G_MASK | DESC_B_MASK | DESC_P_MASK | DESC_S_MASK |
                 (3 << DESC_DPL_SHIFT) | (0xa << DESC_TYPE_SHIFT));
        write_dt(&gdt_table[__USER_DS >> 3], 0, 0xfffff,
                 DESC_G_MASK | DESC_B_MASK | DESC_P_MASK | DESC_S_MASK |
                 (3 << DESC_DPL_SHIFT) | (0x2 << DESC_TYPE_SHIFT));
    }
    cpu_x86_load_seg(env, R_CS, __USER_CS);
    cpu_x86_load_seg(env, R_SS, __USER_DS);
    cpu_x86_load_seg(env, R_DS, __USER_DS);
    cpu_x86_load_seg(env, R_ES, __USER_DS);
    cpu_x86_load_seg(env, R_FS, __USER_DS);
    cpu_x86_load_seg(env, R_GS, __USER_DS);
    /* This hack makes Wine work... */
    env->segs[R_FS].selector = 0;
```

### 15. 进入主循环

```c
    cpu_loop(env);
```