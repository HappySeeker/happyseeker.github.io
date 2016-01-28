---
layout: post
title:  "DRM后端Cairo-perf-trace工具崩溃Again&Again"
date:   2016-01-28 06:06:09
author: Jiang Biao
categories: Graphic
---

# 问题

欲测试cairo DRM后端的性能。 

解决完radeon显卡drm backend中`radeon_surface_backend`结构体中所有错位的钩子后，重新编译运行，发现继续崩溃，但更进一步了。

这次仍然直接core，没有提示信息，查看搜集到的core文件，现象如下：
	(gdb) bt
	#0  0x00000000000000b0 in ?? ()
	#1  0x000000fff5363084 in cairo_drm_surface_map_to_image () from /lib64/libcairo.so.2
	#2  0x000000012000b054 in _cairo_boilerplate_drm_synchronize.lto_priv.66 (closure=0x1205ea280) at cairo-boilerplate-drm.c:69
	#3  0x0000000120007614 in cairo_perf_timer_stop () at cairo-perf.c:73
	#4  cairo_perf_trace.lto_priv.89 (perf=0xffffce69c0, target=0x120024670 <targets.lto_priv>, trace=0xffffce73fc "../trimmed-cairo-traces-master/benchmark/t-gnome-terminal-vim")
	    at cairo-perf-trace.c:776
	#5  0x0000000120004df4 in main (argc=<optimized out>, argv=<optimized out>) at cairo-perf-trace.c:1032

这次在`cairo_perf_timer_stop()`函数中出了问题，段错误。

# 分析

## 现场分析
这次故障出错的地址比较奇怪：

	0x00000000000000b0

肯定是找不到符号信息的，是堆栈有问题？

当然也不是，其实出现段错误就是因为这个地址不对，这个显然是调用某函数时，发现该函数的地址居然是`0x00000000000000b0`，那么就段错误了。

说明在`cairo_drm_surface_map_to_image`函数中，通过指针调用某函数时，出现了空指针，也就是说某函数指针不对。

接下来又只能分析代码了。

## 代码分析

仔细看看`cairo_drm_surface_map_to_image`函数代码：

	/* XXX drm or general surface layer? naming? */
	cairo_surface_t *
	cairo_drm_surface_map_to_image (cairo_surface_t *abstract_surface)
	{
	    cairo_drm_surface_t *surface;
	    cairo_drm_device_t *device;
	    cairo_status_t status;
	
	    if (unlikely (abstract_surface->status))
		return _cairo_surface_create_in_error (abstract_surface->status);
	
	    surface = _cairo_surface_as_drm (abstract_surface);
	    if (surface == NULL) {
		if (_cairo_surface_is_image (abstract_surface))
		    return cairo_surface_reference (abstract_surface);
	
		status = _cairo_surface_set_error (abstract_surface,
						   CAIRO_STATUS_SURFACE_TYPE_MISMATCH);
		return _cairo_surface_create_in_error (status);
	    }
	
	    surface->map_count++;
	    device = (cairo_drm_device_t *) surface->base.device;
	    return cairo_surface_reference (device->surface.map_to_image (surface));  // 问题在这里，surface.map_to_image可能为空指针
	}

看起来，应该是最后一句代码中`map_to_image`为空指针导致的，本想印证下：

	(gdb) bt full
	#0  0x00000000000000b0 in ?? ()
	No symbol table info available.
	#1  0x000000fff5363084 in cairo_drm_surface_map_to_image () from /lib64/libcairo.so.2
	No symbol table info available.
	#2  0x000000012000b054 in _cairo_boilerplate_drm_synchronize.lto_priv.66 (closure=0x1205ea280) at cairo-boilerplate-drm.c:69
	        image = <optimized out>
	#3  0x0000000120007614 in cairo_perf_timer_stop () at cairo-perf.c:73
	No locals.
	#4  cairo_perf_trace.lto_priv.89 (perf=0xffffce69c0, target=0x120024670 <targets.lto_priv>, trace=0xffffce73fc "../trimmed-cairo-traces-master/benchmark/t-gnome-terminal-vim")
	    at cairo-perf-trace.c:776
	        csi = 0x1217bf470
	        status = <optimized out>
	        line_no = 1
	        first_run = 0
	        i = 0
	        times = 0x12002a5f0
	        paint = 0x12002a668
	        mask = 0x12002a6e0
	        fill = 0x12002a7d0
	        stroke = 0x12002a758
	        glyphs = 0x12002a848

但libcairo.so.2的符号表始终找不到，安装了debuginfo包后，还是不行，没有时间究其原因了，算了。

看起来应该是`surface.map_to_image`没有被正确初始化，或者初始化错误。

接下来需要分析`surface.map_to_image`的初始化流程，代码中不同的surface、device、closure、backend把头的绕晕了，很难理清楚。过程就不在这里描述了，后面有机会再单独写相应的分析文章。

最终确认初始化是在`_cairo_drm_radeon_device_create()`函数中初始化的，但发现当前的代码中，并没有初始化`map_to_image()`钩子，问题就出在这里了。

接下来需要确认AMD的radeon驱动中是否实现了这个钩子，翻看代码后确认还好实现了，要是没有实现的话，就麻烦了，估计得自己写了。

# 解决方案

在`_cairo_drm_radeon_device_create()`函数中，将`map_to_image()`钩子初始化为：`radeon_surface_map_to_image`即可。

修改后重新编译，运行cairo-perf-trace工具，终于能看到drm的测试结果了。


