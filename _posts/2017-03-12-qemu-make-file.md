---
layout: post
title: qemu的MakeFile分析
categories: qemu
tags: qemu
---

* content
{:toc}

## 预备内容 ##

首先要了解一些MakeFile的知识，可以参考下面的两个资料：
[MakeFile的书写规则](https://seisman.github.io/how-to-write-makefile/rules.html)  
[MakeFile中的函数](http://blog.csdn.net/ustc_dylan/article/details/6963248)

其次，在分析qemu的MakeFile之前，我们要明确一下我们编译qemu的流程是：先执行configure脚本生成一个MakeFile，然后再make进行编译。

在执行configure脚本的时候是可以传入一些参数的，我们可以通过传入configure的参数控制最终编译的qemu的target平台、host平台、运行模式等，而根据configure参数的不同，生成的MakeFile也会有所不同

因此为了后文中不会产生歧义，这里明确一下本文中编译的qemu是host平台为arm、target平台为x86的Linux user mode的qemu

## 编译的过程 ##

在这里我们编译qemu的流程大致如下：

终端下进入到qemu源码的根目录，然后执行如下命令  
```bash
./configure --target-list=i386-linux-user --cpu=arm --disable-system --disable-bsd-user --disable-tools
make && make install
```

执行完上述命令，编译即完成，在/usr/bin/下就会产生一个qemu-i386的文件，这就是我们编译出的最终结果

## MakeFile的分析 ##

从上面我们知道了我们最终是编译出了一个叫qemu-i386的文件，那么我们就会想MakeFile中肯定会有一个叫qemu-i386的目标。

而且，我们是在qemu源码的根目录执行了make命令的，那么此时make所执行的MakeFile就应该是qemu源码根目录下的MakeFile。于是我们就明确了我们的第一步行动：尝试在qemu源码根目录下的MakeFile中找到qemu-i386这个目标。

然而事与愿违，阅读qemu根目录下的MakeFile并没有找到叫qemu-i386的目标，那么这个qemu-i386文件是如何编译出来的呢？

### 生成qemu-i386的目标 ###

事实是，qemu源码进行编译后，在源码根目录下会产生一个i386-linux-user文件夹，这个文件夹在编译前是不存在的。该文件夹中存放了一些编译过程中所需的文件，包括编译出的目标文件和一个另外的MakeFile，打开这个文件夹中的MakeFile，我们会找到这样的一句：
```bash
$(QEMU_PROG_BUILD): $(all-obj-y) ../libqemuutil.a ../libqemustub.a
	$(call LINK, $(filter-out %.mak, $^))
```
那么QEMU_PROG_BUILD是什么呢？在该MakeFile的前面我们可以看到：
```bash
QEMU_PROG=qemu-$(TARGET_NAME)
QEMU_PROG_BUILD = $(QEMU_PROG)
```
也就是说QEMU_PROG_BUILD是一个形如qemu-xxx的字符串，这样的格式，刚好和我们所想找的qemu-i386所吻合。并且只要我们知道了TARGET_NAME这个变量的值，就可以知道QEMU_PROG_BUILD具体是什么了

我们在这个MakeFile的前边可以看到include了config-target.mak这个文件，我们打开qemu源码根目录/i386-linux-user/config-host.mak这个文件，其中可以找到这么一句：
```bash
TARGET_NAME=i386
```

于是恍然大悟，原来QEMU_PROG_BUILD就是qemu-i386，我们要找的目标就在这里

而这里产生qemu-i386是用了这样的一个语句：
```bash
$(call LINK, $(filter-out %.mak, $^))
```

参考文章开头预备内容中的资料我们知道，其中call和filter-out是MakeFile里边的函数，而LINK是一个定义的变量（LINK变量的内容在下面会分析），这个语句实际是把所有目标文件和库链接到一起产生最终的qemu-i386

### 控制编译的一些变量 ###

在qemu源码根目录/i386-linux-user/MakeFile这个文件的开头，我们可以看到include了其他几个文件：
```bash
include ../config-host.mak
include config-target.mak
include config-devices.mak
include $(SRC_PATH)/rules.mak
```

这里include的前三个文件中都是定义了一些控制编译过程的变量，例如config-target.mak中定义了TARGET_NAME，config-host.mak定义了CC、CFLAGS、LD、LDFLAGS等变量。这些变量都是用来控制编译的过程的，例如我通过修改CFLAGS就可以改变传入编译器的参数，修改CC就可以改变所用的C语言编译器。

而上述这些变量值都是由configure脚本所确定的，我们通过给configure脚本传入参数就可以决定最终生成的这些变量的值。

### 目标文件的生成

在上面include的四个文件中还有一个rules.mak我们没有说到，打开qemu源码根目录下的rule.mak我们可以看到这样的内容：
```bash
%.o: %.S
	$(call quiet-command,$(CCAS) $(QEMU_INCLUDES) $(QEMU_CFLAGS) $(QEMU_DGFLAGS) $(CFLAGS) -c -o $@ $<,"CCAS","$(TARGET_DIR)$@")

%.o: %.cc
	$(call quiet-command,$(CXX) $(QEMU_INCLUDES) $(QEMU_CXXFLAGS) $(QEMU_DGFLAGS) $(CFLAGS) $($@-cflags) -c -o $@ $<,"CXX","$(TARGET_DIR)$@")

%.o: %.cpp
	$(call quiet-command,$(CXX) $(QEMU_INCLUDES) $(QEMU_CXXFLAGS) $(QEMU_DGFLAGS) $(CFLAGS) $($@-cflags) -c -o $@ $<,"CXX","$(TARGET_DIR)$@")
```

这些代码就是用来生成.o目标文件的命令

不仅如此，在前面生成qemu-i386的命令中有一个LINK变量我们没有分析，而这个LINK变量的内容就定义在这个文件中：
```bash
LINKPROG = $(or $(CXX),$(CC))
LINK = $(call quiet-command, $(LINKPROG) $(QEMU_CFLAGS) $(CFLAGS) $(LDFLAGS) -o $@ \
       $(call process-archive-undefs, $1) \
       $(version-obj-y) $(call extract-libs,$1) $(LIBS),"LINK","$(TARGET_DIR)$@")
```
从变量名上我们就可以看出这个LINK变量就是用来把目标文件链接到一起的命令













