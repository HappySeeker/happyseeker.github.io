---
layout: post
title:  "关机动画打开后关机死机问题"
date:   2016-06-06 06:35:05
author: HappySeeker
categories: Graphic
---

# 背景

最近在国产芯片环境中，发现打开开机动画后，关机时，时有死机现象出现，X86环境中没有出现，又是死机问题，由于之前研究过开机动画，这个问题看起来好像可以信手拈来，但过程比预想中要麻烦许多。

# 故障现象

打开开机动画后，执行reboot命令，很快死机，关机动画没有显示出来，显示器黑屏，鼠标键盘没有任何反应，但单板还处于上电状态，不会下电。

故障现象很简单，但要分析就很麻烦了，最大的困难在于没有任何错误打印，打开plymouth的调试开关(在内核启动参数中加入plymouth:debug)，plymouth的日志中也没有任何打印（话说回来，由于是在关机过程中出现问题，即使有日志。启动时也会相应的日志冲掉，所以，不做修改的话，也不能指望这里会有收获。）

另一方面，死机后，不能做任何操作，只能掉电重启，dmesg中也不会有任何收获。

# 故障分析思路

没有错误打印，就没有分析方向，面对这种死机问题，可能会觉得迷茫。但也不是没有办法，个人的思路为：

1. 想办法制造打印。需要分析关机的流程，再分析plymouth在关机过程中的相关流程，在一些关键的地方插桩、打点，增加信息。
2. 最原始的方法。单步调试，总能找到导致死机的地方。

# 故障分析过程

## 分析关机代码和流程

关机的相关主要流程为：

	- 用户执行关机(或重启)操作
	- systemd执行reboot.target相关的一系列操作，主要包括：
	    关闭非reboot.target相关的服务(先发送15、再发送9信号)，包括图形界面相关的进程(都  在lightdm的服务中)
        执行reboot.target相关的服务，包括plymouth.reboot服务。
        真正的关机(重启)操作

##分析plymouth.reboot服务

(位于/usr/lib/systemd/system/目录)，其内容如下：

	[root@localhost ~]# cat /usr/lib/systemd/system/plymouth-reboot.service 
	[Unit]
	Description=Show Plymouth Reboot Screen
	After=getty@tty1.service display-manager.service plymouth-start.service
	Before=systemd-reboot.service
	DefaultDependencies=no
	ConditionKernelCommandLine=!plymouth.enable=0
	
	[Service]
	ExecStart=/usr/sbin/plymouthd --mode=shutdown --attach-to-session
	ExecStartPost=-/usr/bin/plymouth show-splash
	Type=forking
	[Install]
	WantedBy=reboot.target

可以看出，其主要做了两件事：

- /usr/sbin/plymouthd --mode=shutdown --attach-to-session ，用于启动plymouth的服务端后台进程plymouthd(开机后原有的后台进程会quit)，模式设置为shutdown，关于该进程的详细说明，请参考我的另一篇文章。
- /usr/bin/plymouth show-splash，plymouth客户端，用于向服务端发送show-splash命令，请求其显示动画。

## 替换plymouth.reboot服务中的两个操作

将plymouth.reboot服务中的两个操作替换成自己的脚本，并在其中加入--debug --debug-file选项、延时、以及部分信息打印，都没有打印出有效信息，唯一能确认的是，死机发生在第二步过程中（/usr/bin/plymouth show-splash）。
     
## 手工依次执行plymouth.reboot服务中的两个操作

手工执行，发现在执行/usr/bin/plymouth show-splash后，终端中最后的打印信息如下：

	...
	activate:Redrawing screen

## 分析plymouth源码

发现这个打印是在使用Framebuffer驱动时打印的，而正常情况下plymouth应该是使用drm驱动才对，不会使用Framebuffer驱动，这是个重要疑点。

## 没有使用drm驱动的原因

### plymouth相关流程分析

相关的代码流程如下：

	on_show_splash
	  check_for_consoles
	    add_default_displays_and_keyboard
		  ply_renderer_open
			ply_renderer_open_plugin

on_show_splash函数是plymouthd服务端收到show-splash请求时的处理钩子，发送show-splash正是关机时plymouth.reboot服务做的第二个操作。

`ply_renderer_open`函数代码如下：

	bool
	ply_renderer_open (ply_renderer_t *renderer)
	{
	  int i;
	
	  /* FIXME: at some point we may want to make this
	   * part more dynamic (so you don't have to edit this
	   * list to add a new renderer)
	   */
	  const char *known_plugins[] =
	    {
	      PLYMOUTH_PLUGIN_PATH "renderers/x11.so",
	      PLYMOUTH_PLUGIN_PATH "renderers/drm.so",
	      PLYMOUTH_PLUGIN_PATH "renderers/frame-buffer.so",
	      NULL
	    };
	
	  if (renderer->plugin_path != NULL)
	    {
	      return ply_renderer_open_plugin (renderer, renderer->plugin_path);
	    }
	...
	}

其中，`PLYMOUTH_PLUGIN_PATH`为/usr/lib64/plymouth/。可以看出，主要是依次找/usr/lib64/plymouth/renderers/目录中的3个库，先找到那个，就用哪个。

默认情况下，没有x11.so，只有drm和Framebuffer相关的两个库。按顺序，肯定是先用drm，如果drm失败，再用Framebuffer。

当前的问题环境中，应该是使用drm过程中出现了错误导致。但具体是哪里出错仍说不清楚，因为缺乏相应的信息。

接下来，只能单步调试了。重新编译代码，打开调试开关，单步调试后，确认是在执行drmOpen()函数过程中出现了错误，返回失败。

drmOpen是drm驱动提供的用户态接口，在libdrm库中，要确认原因，又必须深入分析libdrm的相关实现了。

### libdrm分析

相关的代码流程如下(接上述plymouth相关流程)：

	ply_renderer_open_plugin
	  ply_renderer_open_device
		open_device
		  load_driver
			drmOpen
			  drmOpenWithType
				drmOpenByName
				  drmOpenMinor

关键函数`drmOpenMinor`实现如下：

	/**
	 * Open the DRM device
	 *
	 * \param minor device minor number.
	 * \param create allow to create the device if set.
	 *
	 * \return a file descriptor on success, or a negative value on error.
	 * 
	 * \internal
	 * Calls drmOpenDevice() if \p create is set, otherwise assembles the device
	 * name from \p minor and opens it.
	 */
	static int drmOpenMinor(int minor, int create, int type)
	{
	    int  fd;
	    char buf[64];
	    const char *dev_name;
	    
	    if (create)
		return drmOpenDevice(makedev(DRM_MAJOR, minor), minor, type);
	    
	    switch (type) {
	    case DRM_NODE_PRIMARY:
		    dev_name = DRM_DEV_NAME;
		    break;
	    case DRM_NODE_CONTROL:
		    dev_name = DRM_CONTROL_DEV_NAME;
		    break;
	    case DRM_NODE_RENDER:
		    dev_name = DRM_RENDER_DEV_NAME;
		    break;
	    default:
		    return -EINVAL;
	    };
	
	    sprintf(buf, dev_name, DRM_DIR_NAME, minor);
	    if ((fd = open(buf, O_RDWR, 0)) >= 0)
		return fd;
	    return -errno;
	}

根据代码可以看出，drmOpen最终是通过open /dev/dri/card0字符设备实现的，open后返回相应的fd。

关键函数`drmOpenByName`的实现如下：
	
	/**
	 * Open the device by name.
	 *
	 * \param name driver name.
	 * \param type the device node type.
	 * 
	 * \return a file descriptor on success, or a negative value on error.
	 * 
	 * \internal
	 * This function opens the first minor number that matches the driver name and
	 * isn't already in use.  If it's in use it then it will already have a bus ID
	 * assigned.
	 * 
	 * \sa drmOpenMinor(), drmGetVersion() and drmGetBusid().
	 */
	static int drmOpenByName(const char *name, int type)
	{
	    int           i;
	    int           fd;
	    drmVersionPtr version;
	    char *        id;
	    int           base = drmGetMinorBase(type);
	
	    if (base < 0)
	        return -1;
	
	    /*
	     * Open the first minor number that matches the driver name and isn't
	     * already in use.  If it's in use it will have a busid assigned already.
	     */
	    for (i = base; i < base + DRM_MAX_MINOR; i++) {
		if ((fd = drmOpenMinor(i, 1, type)) >= 0) {
		    if ((version = drmGetVersion(fd))) {
			if (!strcmp(version->name, name)) {
			    drmFreeVersion(version);
			    id = drmGetBusid(fd);
			    drmMsg("drmGetBusid returned '%s'\n", id ? id : "NULL");
			    if (!id || !*id) {
				if (id)
				    drmFreeBusid(id);
				return fd;
			    } else {
				drmFreeBusid(id);
			    }
			} else {
			    drmFreeVersion(version);
			}
		    }
		    close(fd);
		}
	    }
	...

根据代码和注释，可以看出，drmOpen主要是打开drmOpenMinor打开设备并得到fd后，需要通过drmGetBusid获取busid，如果busid不为空，说明该drm设备已经被其他进程打开过(占有)了，此时为避免冲突，不能继续使用，返回失败。

在什么场景下会被其他进程占用？在你的环境中执行如下命令看看吧：
	
	[root@localhost ~]# lsof|grep card0
	Xorg.bin   941         root  mem       CHR              226,0                 1683 /dev/dri/card0
	Xorg.bin   941         root   10u      CHR              226,0       0t0       1683 /dev/dri/card0
	Xorg.bin   941 1002    root  mem       CHR              226,0                 1683 /dev/dri/card0
	Xorg.bin   941 1002    root   10u      CHR              226,0       0t0       1683 /dev/dri/card0

明显可以看出，我的环境中，Xorg进程占用了该drm设备。分析下Xorg的相关代码，会发现在Xorg启动过程中，在初始化KMS时，会打开该drm设备，并设置其为master，并set相应的busid，相应的代码流程如下，这里就不详细描述了，感兴趣的同学可以自己看看代码：
	
	main
	  dix_main
		InitOutput
	      ->PreInit
			RADEONPreInit_KMS
			  radeon_open_drm_master
				drmSetInterfaceVersion
				  drmIoctl(fd, DRM_IOCTL_SET_VERSION, &sv)
	----------进入内核态-------------------
					drm_setversion
					  drm_set_busid

当然，还有一些其他流程，也会设置这个busid，而且实际过程会比上面描述的更复杂，包括一些dri接口版本的问题，这里也不详细描述了。

### 没有使用drm驱动的原因

综上，没有使用drm驱动的原因是：在关机(或重启)时，有进程占用了drm设备，还没有关闭。

但关机时，不是会先停止服务、kill进程么，为什么还有进程占用drm设备呢？

原因是，桌面系统关机速度很快，停服务、kill进程都不能保证成功，即使成功，也可能不能及时，比如当进程处于D状态时，根据加入的调试和打印信息，这种情况还是普遍存在的。此时drm就不能为plymouth使用，此时关机动画只能使用Framebuffer了。

## 使用Framebuffer时为何死机？

这个问题时本故障的核心问题，按理说，即使drm不能使用，用fb驱动也应该没有问题才对。

### Framebuffer相关原理

Framebuffer对应于屏幕上显示图像在内存中的缓存区域，有关Framebuffer的相关说明，请参考我的其他文章说明。

fb驱动是内核中实现的一个用于直接操作Framebuffer的驱动，简单原理为：将Framebuffer对应的区域映射为一个设备，如/dev/fb0，当用户需要向显示器上绘制图形时，可以直接向该fb设备中写入相应的内容，即可将内容显示到屏幕上。

具体操作时，通常的步骤为：

- 对/dev/fb0设备进行mmap操作，将该设备对应的文件内容映射到进程的虚拟地址空间中。当然这里的mmap操作是由内核的fb驱动实现的。
- 将需要显示的图形对应的数据直接memcpy到上一步映射的虚拟地址中

就这边简单两步即可。

### plymouth使用Framebuffer的相关流程

plymouth也是这么使用的，相关的代码流程如下：

	on_show_splash  
	  show_splash_screen(two-steps)
	    start_progress_animation
	      view_start_progress_animation
	    	ply_progress_animation_show
	    	  ply_progress_animation_draw
				ply_pixel_display_draw_area
				  ply_pixel_display_flush	
					ply_renderer_flush_head
					  ply_renderer_map_to_device
						mmap
					  flush_head
						flush_area
						  flush_area_to_xrgb32_device
							memcpy

流程还比较复杂，关键的两点，在`ply_renderer_flush_head`中先会调用mmap分配(映射)fb中的内存，然后会通过`flush_head`调用`memcpy`将数据拷贝到Framebuffer中。

我的环境中的内存分布情况：

	[root@localhost ~]# cat /proc/597/maps 
	00400000-00413000 r-xp 00000000 08:02 406925                             /usr/sbin/plymouthd
	00422000-00424000 rw-p 00012000 08:02 406925                             /usr/sbin/plymouthd
	00424000-00445000 rw-p 00000000 00:00 0                                  [heap]
	00445000-004c8000 rw-p 00000000 00:00 0                                  [heap]
	7faf4cf000-7faf9cf000 -w-s 00000000 00:06 1119                           /dev/fb0

7faf4cf000-7faf9cf000这个范围对应的大小为1280*1024*4，也就是1280*1024分辨率下的空间。

### 最终原因

通过之前获取到的死机时的打印信息，然后单步跟踪，确认是在执行memcpy函数时，发生了死机。

memcpy为什么死机？这是个复杂的问题，简单说，还是硬件问题，因为memcpy的目的地址对应于Framebuffer，而mmap这段内存时，会通过fb_pgprotect()接口设置该段内存的属性，而该款国产芯片硬件设计有问题，不能使用高级功能，需要特别的设置才行。这里就不详细说明了。


# 总结

此类问题没有任何异常打印，分析定位艰难，需要一步一个脚印。

实际的分析和定位过程远比这里写的复杂，还需要慢慢体会，这个过程中会学到很多东西。



