---
layout: post
title: 编译Android平台上的qemu
categories: qemu
tags: Android qemu C/C++
---

* content
{:toc}

## 搭建编译环境

### 编译环境

首先下载AndroidNDK（只需要AndroidNDK即可，不需要AndroidSDK，也不需要AndroidStudio）

然后按照  
[Linux下搭建Android交叉编译环境](http://blog.fallenworld.org/2017/02/05/android-cross-compile/)    
里的步骤搭建出Android交叉编译环境

### 软件依赖

正式开始编译之前还需要安装必要的一些的软件依赖：
```bash
sudo apt-get install automake texinfo libffi-dev libpcre3-dev
```
libpcre-dev如果用apt安装不了的话，直接从源码安装即可

## 编译依赖库

qemu的编译还依赖于一些其他的库，因此在编译android qemu之前，我们要先把这些依赖的库也在android上编译出来

我们要编译的依赖库有：libpng libiconv libffi pcre gettext glib 

我们在这里回忆一下，我们第一步搭建编译环境时写过一个名为android-build.sh的脚本，而这个脚本就是接下来进行编译的一个核心，我们编译的过程主要就是通过执行这个脚本来编译

### 编译libpng

到libpng的官网（ http://www.libpng.org/pub/png/libpng.html ）下载源码并解压，注意一下libpng-1.6.30的源码有bug，因此应下载1.6.31及以上版本的

将之前提到的android-build.sh复制到libpng的源码的根目录，然后在源码根目录打开终端并执行./android-build.sh，编译即可完成

由此可以见，我们编译的方法实际上十分简单，就是执行了一下android-build.sh脚本便可完成编译

### 编译libiconv

同上，到libiconv官网（ https://www.gnu.org/software/libiconv ）下载源码并解压，复制android-build.sh到源码根目录，终端里执行android-build.sh即可完成编译
   
### 编译libffi

到libffi官网（ https://sourceware.org/libffi ）下载源码并解压，复制android-build.sh到源码根目录，终端里执行android-build.sh即可完成编译

编译完成后，把\$SYSROOT/usr/lib/libffi-3.2.1/include下的所有头文件复制到\$SYSROOT/usr/include下（若编译的libffi的版本号是其他版本的，则把上述路径中的3.2.1换成相应的版本号）

### 编译pcre

到pcre官网（ http://www.pcre.org ）下载源码并解压（注意不要下载pcre2），复制android-build.sh到源码根目录，终端里执行android-build.sh即可完成编译

### 编译gettext

到gettext官网（ https://www.gnu.org/software/gettext ）下载源码并解压，复制android-build.sh到源码根目录

注意一下这里和前几个有所不同，此时若直接在终端里执行android-build.sh，则编译过程中会出现若干报错，因此我们先修复这些编译错误后再执行androd-build.sh来编译

**编译问题排错**

1. nl_langinfo.c的case语句中一些宏未定义
   
   打开gettext源码根目录/gettext-tools/libgrep/nl_langinfo.c，在文件开头添加：
   ```c
   #ifdef __ANDROID__
   # define INT_CURR_SYMBOL   10100
   # define MON_DECIMAL_POINT 10101
   # define MON_THOUSANDS_SEP 10102
   # define MON_GROUPING      10103
   # define POSITIVE_SIGN     10104
   # define NEGATIVE_SIGN     10105
   # define FRAC_DIGITS       10106
   # define INT_FRAC_DIGITS   10107
   # define P_CS_PRECEDES     10108
   # define N_CS_PRECEDES     10109
   # define P_SEP_BY_SPACE    10110
   # define N_SEP_BY_SPACE    10111
   # define P_SIGN_POSN       10112
   # define N_SIGN_POSN       10113
   # define GROUPING          10114
   #endif
   ```
   解决方法就是把这些没定义的宏定义一下就行了

2. error: cannot find -lgettextlib

   打开android-build.sh，在LIBDIRS变量的内容中添加-L\$(pwd)/gettext-tools/gnulib-lib/.libs，注意要和前面一个库路径用空格隔开
   
   出现这个错误的原因是libgettextlib.a的路径找不到，因此我们把这个库的路径放到编译的环境变量中即可解决

做好上述修改后，在终端里执行android-build.sh即可编译成功

### 编译glib

编译glib之前要说一下，实际上我们这里编译qemu时所依赖的库只有libpng和glib，qemu自身并不依赖于libiconv libffi pcre gettext。但是glib是依赖于这四个库的，因此我们编译libiconv libffi pcre gettext实际上是为编译glib做准备的。而现在这四个库已经编译好了，我们就可以开始编译glib了

下载glib源码（ https://ftp.gnome.org/pub/gnome/sources/glib/ ）并解压，复制android-build.sh到源码根目录

在glib源码根目录下创建一个名为android.cacahe的文件，内容如下：
```
glib_cv_long_long_format=ll
glib_cv_stack_grows=no
glib_cv_uscore=no
ac_cv_func_posix_getpwuid_r=no
ac_cv_func_posix_getgrgid_r=no
```

（如果编译过程中出现了问需要重新编译，则需要把android.cache的内容重新恢复成上面的内容）

打开刚刚复制到glib源码根目录下的android-build.sh，分别将CFLAGS、CXXFLAGS、LDFLAGS、CONFLAGS的值改为：
```bash
CFLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16 -D\__ANDROID_API\__=\$API_VERSION -fPIC"
CXXFLAGS="-std=c++11 -fPIC"
LDFLAGS="-march=armv7-a -Wl,--fix-cortex-a8 \$LIBDIRS -fPIC"
CONFLAGS="--prefix=\${SYSROOT}/usr --host=\$HOST --cache-file=android.cache --enable-static --disable-libmount"
```

（实际上做出的修改就是在CFLAGS、CXXFLAGS、LDFLAGS中添加了-fPIC，在CONFLAGS中添加了--cache-file=android.cache --enable-static --disable-libmount）

在终端里执行android-build.sh即可编译成功

## 编译qemu

终于到了我们的正餐 —— 编译qemu的时刻，编译qemu的步骤稍显繁琐，但核心的思路其实也很简单：就是通过android-build.sh这个脚本编译，如果编译过程中遇到错误的就针对这个错误进行一些修复

那就让我们开始吧！

到qemu官网（ https://www.qemu.org/download/#source ）下载源码并解压，复制android-build.sh到源码根目录

打开刚刚复制到qemu源码根目录下的android-build.sh，分别将CFLAGS、LDFLAGS、CONFLAGS修改为：

```bash
CFLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16 -D__ANDROID_API__=$API_VERSION -fPIE -pie -fPIC"
LDFLAGS="-march=armv7-a -Wl,--fix-cortex-a8,-soname,libqemu.so $LIBDIRS -fPIE -pie -fPIC -shared"
CONFLAGS="--prefix=${SYSROOT}/usr --cross-prefix=${HOST}- --host-cc=$CC --target-list=i386-linux-user --cpu=arm --disable-system --disable-bsd-user --disable-tools --disable-zlib-test --disable-guest-agent --disable-nettle --enable-debug"
```

并且在android-build.sh中执行configure的那一行之前添加一句：
```bash
ln -s /usr/bin/pkg-config $ANDROID_BUILD/bin/arm-linux-androideabi-pkg-config
```


然后打开qemu源码根目录下的configure文件，找到以下内容：

```bash
if test "$pie" = ""; then
  case "$cpu-$targetos" in
    i386-Linux|x86_64-Linux|x32-Linux|i386-OpenBSD|x86_64-OpenBSD|arm-linux)
      ;;
    *)
      pie="no"
      ;;
  esac
fi
```

将
```bash
    i386-Linux|x86_64-Linux|x32-Linux|i386-OpenBSD|x86_64-OpenBSD)
```
这一行改为
```bash
    i386-Linux|x86_64-Linux|x32-Linux|i386-OpenBSD|x86_64-OpenBSD|arm-linux)
```

进行这样的修改的原因可以参考本文最后的参考资料

**编译问题排错**

1. ld: error: cannot find -lutil

   将qemu源码根目录下的Makefile文件中下面的内容注释掉（单行注释用#）
   ```bash
   ifneq ($(wildcard config-host.mak),)
   include $(SRC_PATH)/tests/Makefile.include
   endif
   ```

2. error: mqueue.h: No such file or directory

   打开qemu源码根目录/linux-user/syscall.c，找到
   ```c
   #include <mqueue.h>
   ```
   将其改为
   ```c
   #include <linux/mqueue.h>
   ```
   
3. error: static declaration of 'gettid' follows non-static declaration

   打开qemu源码根目录/linux-user/syscall.c，将
   ```c
   #ifdef __NR_gettid
   _syscall0(int, gettid)
   #else
   /* This is a replacement for the host gettid() and must return a host
      errno. */
   static int gettid(void) {
       return -ENOSYS;
   }
   #endif
   ```
   改为
   ```c
   #ifndef __ANDROID__
   #ifdef __NR_gettid
   _syscall0(int, gettid)
   #else
   /* This is a replacement for the host gettid() and must return a host
      errno. */
   static int gettid(void) {
       return -ENOSYS;
   }
   #endif
   #endif
   ```

4. error: 'struct ipc64_perm' has no member named '\__key'  
   error: 'struct ipc64_perm' has no member named '\__seq'
   error: 'struct msqid64_ds' has no member named '\__msg_cbytes'
  
   打开qemu源码根目录/linux-user/syscall.c，将所有的
   ```c
   host_ip->__key
   ```
   替换为
   ```c
   host_ip->key
   ```
   同理，将所有的
   ```c
   host_ip->__seq
   ```
   替换为
   ```c
   host_ip->seq
   ```
  以及，将所有的
   ```c
   host_md->__msg_cbytes
   ```
   替换为
   ```c
   host_md->msg_cbytes
   ```
5. error: redefinition of 'union semun'
   
   打开qemu源码根目录/linux-user/syscall.c，将
   
   ```c
   union semun {
	   int val;
	   struct semid_ds *buf;
	   unsigned short *array;
	   struct seminfo *__buf;
   };
   ```
   改为
   ```c
   #ifndef __ANDROID__
   union semun {
	   int val;
	   struct semid_ds *buf;
	   unsigned short *array;
	   struct seminfo *__buf;
   };
   #endif
   ```
   
6. error: dereferencing pointer to incomplete type  
   host->c_iflag =
   
   打开qemu源码根目录/linux-user/syscall.c，将
   
   ```c
   #define termios host_termios
   ```
   改为
   ```c
   #ifndef __ANDROID__
   #define termios host_termios
   #else
   #define host_termios termios 
   #endif
   ```
   
做好上述修改后，在终端里执行android-build.sh即可编译成功
  
编译完成后，打开\$ANDROID_BUILD/sysroot/usr/bin，在该目录下即可看到编译出的一个qemu-i386文件，将其重命名为libqemu.so，就可以把它当作一个一般的so库，在你的Android应用中通过jni加载这个动态库并调用里面的函数了

到此为止大功告成！
  
## 后记

### 本文中的实验环境

Linux Mint 18.1 Serena (Ubuntu 16.04)  
Android NDK r14b  
libpng-1.6.31  
libiconv-1.15  
libffi-3.2.1  
pcre-8.41  
gettext-0.19.8.1  
glib-2.52.3  
qemu-2.9.0

### 参考资料

编译可在Android上运行的qemu user mode: https://my.oschina.net/ibuwai/blog/649670





