---
layout: post
title: 编译Android平台上的qemu
categories: qemu
tags: Android qemu C/C++
---

* content
{:toc}

安装需要的软件依赖：
```bash
sudo apt-get install automake texinfo libffi-dev libpcre3-dev
```
---

在$(ANDROID_BUILD)/sysroot/usr/include/sys/mman.h中搜索mremap函数的声明，其原本的函数声明如下：
```c
extern void* mremap(void*, size_t, size_t, unsigned long);
```
将其改为：
```c
extern void* mremap(void*, size_t, size_t, unsigned long, ...);
```

---
## 

在glib源码目录/glib/gmessages.c文件中搜索：
```c
if ((buf_fd = mkostemp (path, O_CLOEXEC|O_RDWR)) < 0)
```
将这一句代码修改为
```c
#ifdef __ANDROID__
  if ((buf_fd = mkstemp (path)) < 0)
#else
  if ((buf_fd = mkostemp (path, O_CLOEXEC|O_RDWR)) < 0)
#endif
```
若不进行修改的话，在编译时会报错：
```bash
gmessages.c: In function 'journal_sendv':
gmessages.c:2144:3: error: implicit declaration of function 'mkostemp'
```
报错原因：Android NDK中没有mkostemp这个函数  
解决方法：用mkstemp函数来代替mkostemp函数








