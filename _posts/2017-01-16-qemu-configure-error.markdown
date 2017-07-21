---
layout: post
title:  "Qemu static方式configure错误"
date:   2017-01-11 06:10:54
author: HappySeeker
categories: Virt
---

# 现象

需要编译qemu，使用static的方式，带--static参数执行configure，报错：

    [root@localhost qemu]# ./configure --static
    big/little test failed

    Error: zlib check failed
    Make sure to have the zlib libs and headers installed.

# 尝试

使用yum安装了所有zlib相关的包，包括i686和x86_64的，还是不行。

修改configure文件，跳过zlib的检测，然后又报新错：

    [root@localhost qemu]# ./configure --target-list=i386-linux-user --static
    big/little test failed

    Error: pthread check failed
    Make sure to have the pthread libs and headers installed.

pthread库都不找不到，不太可能吧？

搜了下，可能是静态编译的问题，需要安装依赖库的静态包，尝试安装了glib2和zlib的static包，还是不行，郁闷了。

    [root@localhost qemu]# rpm -qa|grep glib2
    glib2-devel-2.42.1-1.1.fc21.x86_64
    glib2-2.42.1-1.1.fc21.i686
    glib2-static-2.42.1-1.1.fc21.i686
    pulseaudio-libs-glib2-5.0-25.1.fc21.x86_64
    glib2-static-2.42.1-1.1.fc21.x86_64
    glib2-2.42.1-1.1.fc21.x86_64
    glib2-devel-2.42.1-1.1.fc21.i686
    [root@localhost qemu]# rpm -qa|grep zlib
    zlib-devel-1.2.8-7.fc21.x86_64
    zlib-devel-1.2.8-7.fc21.i686
    zlibrary-ui-qt-0.12.10-16.fc21.x86_64
    zlib-static-1.2.8-7.fc21.i686
    zlibrary-0.12.10-16.fc21.x86_64
    zlib-1.2.8-7.fc21.x86_64
    zlib-1.2.8-7.fc21.i686
    zlibrary-devel-0.12.10-16.fc21.x86_64
    zlib-static-1.2.8-7.fc21.x86_64
    zlib-ada-devel-1.4-0.8.20120830CVS.fc21.x86_64
    zlibrary-ui-gtk-0.12.10-16.fc21.x86_64
    zlib-ada-1.4-0.8.20120830CVS.fc21.x86_64

# 解决

最后，还是在google中找到了答案：

    it is a bug of qemu.in fact,it is the glibc is not installed

看看glibc装了没有：

    [root@localhost qemu]# rpm -qa|grep glibc
    glibc-headers-2.20-5.fc21.x86_64
    glibc-2.20-5.fc21.x86_64
    glibc-devel-2.20-5.fc21.i686
    glibc-2.20-5.fc21.i686
    glibc-devel-2.20-5.fc21.x86_64
    glibc-common-2.20-5.fc21.x86_64

static包确实没装，安装后问题解决

    yum install glibc*

# Why？

不要满足于解决现有问题，继续看看configure中相关检测代码：

    if test "$zlib" != "no" ; then
        cat > $TMPC << EOF
    #include <zlib.h>
    int main(void) { zlibVersion(); return 0; }
    EOF
        if compile_prog "" "-lz" ; then
            :
        else
            echo
            echo "Error: zlib check failed"
            echo "Make sure to have the zlib libs and headers installed."
            echo
            exit 1
        fi
    fi

其实，就是写了一个简单的使用zlib的程序，然后尝试编译，编译不通过就Error，原理看似每问题，但其实不够准确，导致编译失败的原因可能不止于zlib的问题，最基础的glibc出问题也会这样～
