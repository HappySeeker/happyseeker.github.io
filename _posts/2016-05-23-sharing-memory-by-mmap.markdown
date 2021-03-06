---
layout: post
title:  "使用mmap来共享内存"
date:   2016-05-3 6:13:11
author: HappySeeker
categories: Kernel
---

# 背景

Linux中常使用mmap来实现共享内存，但场景还是比较多，这里做简单介绍。

# 什么是mmap？

mmap的本质是用来将文件(也可以是设备)内容映射到进程地址空间(虚拟内存，返回虚拟内存地址)，映射后，进程直接往指定内存地址(虚拟内存地址)中读写内容，即可实现该文件内容的读写。

这里有几种典型的使用场景：

1. 当mmap中使用的文件描述符(fd)为空时，称为“匿名映射”，此时mmap的本质为分配内存(看起来跟mmap的名字不符)，作用类似于malloc(实际上，malloc的部分流程就是通过mmap匿名映射实现的)。简单说，就是分配一段内存，返回该内存的其实地址(这里的内存都是虚拟内存，物理内存通过缺页异常实现)。
2. 当使用普通文件的描述符时，这个就是最常见的场景了，作用如之前所述。
3. 当使用特殊文件的描述符时，比如使用内存设备文件，此时实际是将内存映射到进程的地址空间中，这种场景多用于实现共享内存。

# 用mmap实现共享内存的场景

实际上，用mmap来实现共享内存也有不同的场景，典型场景如下：

1. 用户态进程间使用磁盘文件来实现内存。典型实现方式为：两个进程都使用mmap映射文件(或磁盘设备)，这里相同的文件内容同时对两个进程都可见，进程各自向其映射的地址中读写内容，即可实现共享内存。由于使用磁盘，比较慢，效率低。
2. 父子进程间共享内存。典型实现方式为：父进程中使用mmap匿名映射一块内存(相当于是分配了一段虚拟内存)，然后fork子进程，由于子进程会默认继承父进程的地址空间，所以子进程中也能看到父进程之前mmap映射的虚拟地址范围，那么父子进程之间就可以通过这段内存(虚拟地址空间)通信了。
3. 用户态和内核直接共享内存。典型实现步骤为：
	- 在内核态分配物理内存，然后映射到内核地址空间(线性映射)，内核使用映射的虚拟地址进行通信(读写共享内存)
	- 将内核分配内存的物理内存信息(起始地址、大小)写入/proc中
	- 在内核态创建虚拟设备(比如一个简单的字符设备)，用于在用户态中mmap中使用。
	- 用户态open虚拟设备
	- 用户态通过/proc读取之前内核写入的物理内存信息
	- 基于打开设备的fd、从/proc中读取的物理内存信息，使用mmap来映射，得到用户态的虚拟地址。
	- 用户态使用该虚拟地址对应的内存区域与内核通信。
