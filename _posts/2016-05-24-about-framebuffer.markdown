---
layout: post
title:  "闲聊Framebuffer"
date:   2016-05-24 06:15:07
author: HappySeeker
categories: Kernel
---

# 背景

接触过图形相关的同学应该对Framebuffer这个名词不陌生，但Framebuffer究竟是什么，用来做什么，在我接触图形相关工作以前，对我来说一直是模糊的。

本文主要闲聊Framebuffer。

# 什么是Framebuffer？

Framebuffer，也叫帧缓冲，其内容对应于屏幕上的界面显示，可以将其简单理解为屏幕上显示内容对应的缓存，修改Framebuffer中的内容，即表示修改屏幕上的内容，所以，直接操作Framebuffer可以直接从显示器上观察到效果。

但Framebuffer并不是屏幕内容的直接的像素表示。Framebuffer实际上包含了几个不同作用的缓存，比如颜色缓存、深度缓存等，具体不详细说明。大家只需要知道，这几个缓存的共同作用下，形成了最终在屏幕上显示的图像。

Framebuffer本质上是一段内存，哦，不对，是一段显存，哦，好像还是不对。

其实，Framebuffer就是一段存储空间，其可以位于显存，也可以位于内存。

Framebuffer是一个逻辑上的概念，并非在显存或者是内存上有一块固定的物理区域叫Framebuffer。实际上，物理是显存或者内存，只要是在GPU能够访问的空间范围内(GPU的物理地址空间)，任意分配一段内存(或显存)，都可以作为Framebuffer使用，只需要在分配后将该内存区域信息，设置到显卡相关的寄存器中即可。这个其实跟DMA区域的概念是类似的。

# 如何使用Framebuffer？

如前面所说，Framebuffer是可以位于显存或内存上分配的任意内存，从驱动的角度说，这块内存对应于bo(buffer object)，使用具体驱动提供的接口进行分配，分配时可以通过参数指定从内存(GTT)或显存(VRAM)上分配，分配后，直接修改显卡相关的寄存器配置即可。

具体来说，就是在分配缓存后，使用drm驱动提供的接口设置，以raedon驱动为例，分配的缓存对应于radeon_bo结构，代码示例为：

	struct radeon_bo *buffer_object;
	...
	buffer_object = radeon_bo_open (driver->manager, 0,
                                  height * *row_stride,
                                  0, RADEON_GEM_DOMAIN_GTT, 0);
	...
	if (drmModeAddFB (driver->device_fd, width, height,
                    24, 32, *row_stride, buffer_object->handle,
                    &buffer_id) != 0)
    {
      ply_trace ("Could not set up GEM object as frame buffer: %m");
      radeon_bo_unref (buffer_object);
      return 0;
    }
	...

调用的两个关键接口：

- radeon_bo_open：分配缓存，本例中指定了从内存(GTT)中分配
- drmModeAddFB：设置Framebuffer，具体由底层drm驱动实现，这里不讨论

# GTT VS. VRAM

这两个名词不知大家是否听说过，其实很简单，GTT就是在内存(主存)上分配的内存区域，VRAM即在显存上分配的内存区域，这里不聊名词的来源和意义，只简单聊一下使用GTT和使用VRAM的底层处理逻辑的区别。

## 使用GTT

当使用GTT时，将图像显示到显示器上的大致逻辑是这样的：

1. 在内存中分配一块缓存区域(作为Framebuffer)
2. 将需要绘制的图形对应的数据拷贝到这块内存区域，当然，这个拷贝操作显然是由CPU负责的，即消耗的是CPU，GPU完全不参与。
3. GPU将Framebuffer中内容显示到显示器上(swapBuffer)。这个操作是由GPU负责的，消耗的是GPU，CPU基本不参与。这个过程显然有个数据搬移(可以理解为拷贝)的操作，毕竟搬移之前，数据还在内存中，是不可能直接显示到显示器上的。这个过程可以理解为一次DMA操作。

这个过程可以看出，如果使用GTT在做Framebuffer，存在“两次”数据拷贝操作：

1.将数据拷贝到Framebuffer中
2.从Framebuffer到显示器的数据搬移

虽然有两次数据拷贝操作，但CPU只负责其中一次，另一次由GPU负责。

## 使用VRAM

当使用VRAM时，将图像显示到显示器上的大致逻辑是这样的：

1. 在显存中分配一块缓存区域(作为Framebuffer)，并将其映射到CPU的物理地址空间中。
2. 将需要绘制的图形对应的数据拷贝到这块内存区域，当然，这个操作也是CPU负责的，GPU基本不参与。
3. GPU将Framebuffer中内容显示到显示器上(swapBuffer)。这个操作由GPU负责，但由于Framebuffer已经在显存中，所以这里并不存在数据搬移操作，具体如何显示到显示器上我们就不关心了，反正是显卡硬件搞定的。

从这个过程可以看出，如果使用VRAM做Framebuffer，只存在“一次”数据拷贝操作：

1.将数据拷贝到Framebuffer中

该操作由CPU负责。

对比使用GTT的情况，是否说明使用VRAM做Framebuffer就一定更高效呢(比较少一次内存拷贝操作嘛)？

不一定！为啥？

对比GTT和VRAM，看似两种情况下，第一次内存拷贝操作“将数据拷贝到Framebuffer中”都是一样的，但其实不然，虽然都是内存拷贝操作，但是GTT情况下，从“内存到内存”的拷贝，依赖的是CPU的系统总线，效率更高；而VRAM情况下，是从“内存到显存”的拷贝，依赖的是PCIE总线(当前显卡多是通过PCIE总线连接的)，效率相对较低。

同样都是由CPU负责，显然，在VRAM情况下，其消耗的CPU时间会更长。

而另一方面，在GTT情况下，第二次内存搬移操作虽然由GPU负责完成，类似于DMA操作，但实际上，DMA操作也是需要一定的CPU消耗的，比如上下文切换。

所以，使用VRAM还是GTT做Framebuffer，谁比较高效，这个并不绝对，要看具体场景，还要分不同角度：CPU的角度和GPU的角度。

具体什么场景下，用哪种方式高效，留给大家回去仔细思考吧，这个话题很有意思。

