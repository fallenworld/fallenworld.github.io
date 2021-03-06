---
layout: post
title: Linux下搭建Android交叉编译环境
categories: Android
tags: C/C++ Android Linux
---

* content
{:toc}

## 前言

为了能够在Android平台上使用一些Linux中的C/C++库，我们需要使用AndroidNDK来编译那些Linux库的源代码，使用NDK开发时，通常使用Android.mk或者Cmake来构建C/C++代码

但是一般Linux库是通过一个configure脚本来生成MakeFile的方式来构建的，在Linux上构建一个Linux库的一般流程如下：

```bash
./configure
make
make install
```

这样的话就和我们通常使用NDK时构建C/C++代码的方式不一样

一种常见的解决方法就是把configure生成的MakeFile手动转换为Android.mk或者CmakeList.txt，这种方法在项目规模比较小的时候还比较可行。但是当项目比较庞大，依赖复杂时，就很难去进行转换了，那么我们就想能不能像上面一样直接用configure生成MakeFile来进行编译呢？答案是完全可以，而且这种方法相对于转换MakeFile的方法还更加简单易行

我们基本的思路就是在进行编译的时候，把我们默认用的gcc、g++、ld等编译工具替换成AndroidNDK中所提供的编译工具来进行交叉编译就行了，接下来的内容以在lubuntu 64位系统上为例就讲解了搭建编译环境的过程

参考官方文档：[https://developer.android.com/ndk/guides/standalone_toolchain.html](https://developer.android.com/ndk/guides/standalone_toolchain.html)


## 下载Android NDK

首先到官网下载Android NDK并解压到某一位置

下载地址：[https://developer.android.com/ndk/downloads/index.html](https://developer.android.com/ndk/downloads/index.html)

本文中NDK版本为ndk-r14b-linux-x86_64


## 运行NDK中的环境搭建脚本

终端下进入到AndroidNDK目录/build/tools/下

运行如下命令

```bash
./make_standalone_toolchain.py --arch arm --api 24 --unified-headers --install-dir ~/android-build
```

参数解释：

--arch：交叉编译的目标平台架构，因为我们的Android手机基本都是arm平台，因此这里写arm

--unified-headers：使用libc头文件，相关解释可以参考 https://android.googlesource.com/platform/ndk.git/+/ndk-r14-release/docs/UnifiedHeaders.md

--api：Android系统的版本

--install-dir：生成的交叉编译构建工具的输出位置，这里我把交叉编译工具生成到了~/android-build下，当然你也可以设置成别的路径

这个脚本是Android NDK中官方所提供的脚本，功能就是搭建一个交叉编译环境，脚本的更多参数和详情请参考前言中给出的官方文档


## 编写脚本模版

在Android交叉编译工具文件夹的根目录下新建一个脚本文件android-build.sh，内容如下：

```bash
#!/bin/bash

ANDROID_BUILD=上一步中你生成的Android交叉编译工具的路径
API_VERSION=24
PATH=$ANDROID_BUILD/bin:$PATH
SYSROOT=$ANDROID_BUILD/sysroot
HOST=arm-linux-androideabi
CFLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16 -D__ANDROID_API__=$API_VERSION"
CXXFLAGS="-std=c++11"
LIBDIRS="-L$ANDROID_BUILD/arm-linux-androideabi/lib"
LDFLAGS="-march=armv7-a -Wl,--fix-cortex-a8 $LIBDIRS"
CONFLAGS="--prefix=${SYSROOT}/usr --host=$HOST"
PKG_CONFIG_PATH="$SYSROOT/usr/lib/pkgconfig"

#configure
PKG_CONFIG_PATH=$PKG_CONFIG_PATH CFLAGS="$CFLAGS" CXXFLAGS="$CXXFLAGS" LDFLAGS="$LDFLAGS" ./configure $CONFLAGS &&

#make & install
make && make install

```

这是一个用来编译Linux库的脚本模版，之后我们要为Android编译某个Linux库的时候，在这个脚本模版的基础上进行修改即可

我们可以看到这个脚本中定义了许多变量，这些变量都是一些编译的时候要用到的参数，我们可以看到在脚本中运行configure的时候我们把这些参数传了进去，这样我们就可以通过这些参数指定用我们的交叉编译工具来编译

脚本中的变量解释如下：

ANDROID_BUILD：Android交叉编译工具的路径

PATH：我们把交叉编译工具根目录下的bin文件夹的路径加入了PATH环境变量中，这样我们就可以直接运行bin文件下的那些工具了

SYSROOT：编译时所使用的系统根目录的路径，这里解释下这个系统根目录是什么意思：我们PC上的Linux系统根目录就是/，我们在编译的时候会用到一些系统根目录中的文件，例如我们为PC上的Linux编译一些C/C++代码的时候要用到/usr/include中的头文件，还要用到/usr/lib中的一些库，还可能要执行/bin下的一些程序。但是此时我们是要为Android平台进行交叉编译，此时用到的头文件、库等得是Android平台的，因此我们要使用交叉编译工具提供的系统根目录\$ANDROID_BUILD/sysroot

HOST：编译的目标平台，我们在configure的参数指定了--host=\$HOST（即--host=arm-linux-androideabi）之后，编译时就会使用形如arm-linux-androideabi-XXX的工具，例如编译时使用的gcc就会是arm-linux-androideabi-gcc，使用的g++就会是arm-linux-androideabi-g++。还记得之前我们把\$ANDROID_BUILD/bin这个路径加入到PATH环境变量了吗？arm-linux-androideabi-gcc、arm-linux-androideabi-g++这些编译工具的文件就在这个bin文件中

CFLAGS：gcc所用的参数，为什么要传入那些参数请参考前言中官方文档链接

CXXFLAGS：g++所用的参数，这里指定了要使用c++11标准

LIBDIRS：库文件的路径，这里是把Android的C++STL库的路径加入了库路径

LDFLAGS：链接器ld使用的参数，为什么要传入那些参数请参考前言中的官方文档链接

CONFLAGS：运行configure脚本时要传入的参数，--prefix是指定了make install时的安装路径，--host前面解释过了

PKG_CONFIG_PATH：具体的解释可以参考这里 http://www.cppblog.com/colorful/archive/2012/05/05/173750.aspx

可以看到我们这个脚本中进行编译的过程也是先执行configure，然后make和make install，和一般Linux编译的过程一样，只是需要指定一些平台的参数

之后如果我们为Android编译一些Linux库，只要把这个脚本复制到合适的位置并进行一些简单的修改，然后运行这个脚本就可以了，十分方便
