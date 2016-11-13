---
layout: post
title:  "Wayland源码分析-repaint相关流程"
date:   2016-11-11 6:43:57
author: HappySeeker
categories: Graphic
---


本文关注Wayland中绘图相关流程，这是wayland中非常关键的流程之一。

# repaint？

为什么wayland中要有repaint操作呢？不是号称都是client绘图、wayland只负责合成么？

确实是Client绘图，compositor(服务端)只负责合成。但由于client绘图实际实现为double buffer，client最初的绘图操作都是在pending buffer(也就是我们理解的back buffer)中进行的，并没有直接绘制到framebuffer中，所以，在client完成绘制后，需要进行commit操作(前面的文章已经介绍过)，commit后，服务端会将相应的buffer会更新到当前的surface中，此时，需要进行repaint操作，将客户端相应的绘制内容最终拷贝到framebuffer中。

这里的repaint操作，本质上只是将客户端更新的绘制内容(damage区域)提交给compositor，最后由compositor进行合成后绘制到界面上。

所以，本质上，还是client绘图，compositor进行合成，没有矛盾。

# 流程

以weston实现代码分析，从surface_commit开始到repaint的简要流程为：

    surface_commit
      weston_surface_commit
        weston_surface_schedule_repaint

再继续看看weston_surface_schedule_repaint代码实现：

    WL_EXPORT void
    weston_surface_schedule_repaint(struct weston_surface *surface)
    {
    	struct weston_output *output;

    	wl_list_for_each(output, &surface->compositor->output_list, link)
    		if (surface->output_mask & (1 << output->id))
    			weston_output_schedule_repaint(output);
    }

很简单，只是对output_list中的所有output调用weston_output_schedule_repaint操作。

这里的output对应一个输出设备，比如显示器、远程桌面等，这些output是在compositor的初始化过程中创建的，先看看这个初始化流程。

# output初始化

其中涉及到两个关键的地方：

1. weston支持在不同的backend(后端)上运行，包括drm、wayland和X11。这里需要注意这集中后端的区别，尤其是wayland后端，很容易弄混。

  - drm后端，是wayland环境中默认使用的后端，其使用Linux KMS输出，使用evdev作为输入，实际就是利用drm的接口实现合成。利用了硬件加速。
  - wayland后端，这是指让weston运行于另一个wayland compositor(比如另一个weston，嵌套运行)之上，作为wayland的客户端运行，此时的weston实例表现为另一个compositor上的一个窗口。注意：这个不是默认的方式，这是嵌套方式，使用较少。
  - X11后端，即weston运行在X11之上，每个weston output作为一个X window。这种方式对于weston的测试非常有用(可以在现有的Xorg环境中测试weston的功能)。

2. 使用特定后端的情况下，绘图时，支持使用不同的renderer（绘图引擎），比如OpenGL(EGL,硬件加速)或者时pixman(软件绘制)，使用wayland作为后端时，默认使用OpenGL作为renderer。

这两点的实现，都需要进行抽象、分层，以便于隔离，减少耦合，比较常见的实现手法，函数指针(回调函数)（对应于OO语言（如C++、Java）中的多态）。典型的依赖倒置原则。

对于关键点1的实现还有点不一样(虽然思想是类似的)，其实现为在运行时动态加载共享库，使用其中的接口，简单说，就是用dlopen，然后每个backend实现为相应的动态库。相应流程为：

    main ->
      load_backend ->
        load_drm_backend ->
          weston_compositor_load_backend ->
            weston_load_module

weston_load_module实现为：

    WL_EXPORT void *
    weston_load_module(const char *name, const char *entrypoint)
    {
    	char path[PATH_MAX];
    	void *module, *init;

    	if (name == NULL)
    		return NULL;

    	if (name[0] != '/')
    		snprintf(path, sizeof path, "%s/%s", MODULEDIR, name);
    	else
    		snprintf(path, sizeof path, "%s", name);

    	module = dlopen(path, RTLD_NOW | RTLD_NOLOAD);
    	if (module) {
    		weston_log("Module '%s' already loaded\n", path);
    		dlclose(module);
    		return NULL;
    	}

    	weston_log("Loading module '%s'\n", path);
    	module = dlopen(path, RTLD_NOW);
    	if (!module) {
    		weston_log("Failed to load module: %s\n", dlerror());
    		return NULL;
    	}

    	init = dlsym(module, entrypoint);
    	if (!init) {
    		weston_log("Failed to lookup init function: %s\n", dlerror());
    		dlclose(module);
    		return NULL;
    	}

    	return init;
    }

代码简单直接。

上述关键点2,实现体现在了output的初始化流程中，相关主要流程为：

    main
      load_backend->
        load_drm_backend->
          drm_backend_output_configure->
            weston_output_enable->
              output->enable->
                drm_output_enable

看看drm_output_enable的实现：

    static int
    drm_output_enable(struct weston_output *base)
    {
    	struct drm_output *output = to_drm_output(base);
    	struct drm_backend *b = to_drm_backend(base->compositor);
    	struct weston_mode *m;

    	output->dpms_prop = drm_get_prop(b->drm.fd, output->connector, "DPMS");
      //设置renderer
    	if (b->use_pixman) {
        //使用pixman软件绘制
    		if (drm_output_init_pixman(output, b) < 0) {
    			weston_log("Failed to init output pixman state\n");
    			goto err_free;
    		}
        //使用egl，即openGL接口
    	} else if (drm_output_init_egl(output, b) < 0) {
    		weston_log("Failed to init output gl state\n");
    		goto err_free;
    	}

      ...
      // 设置start_repaint_loop接口，在commit后的repaint流程中调用
    	output->base.start_repaint_loop = drm_output_start_repaint_loop;
      // 设置repaint接口，用于最终绘制
    	output->base.repaint = drm_output_repaint;
    	output->base.assign_planes = drm_assign_planes;
    	output->base.set_dpms = drm_set_dpms;
    	output->base.switch_mode = drm_output_switch_mode;
      ...
    }

# 继续repaint流程

从weston_surface_schedule_repaint开始，

    weston_surface_schedule_repaint->
      weston_output_schedule_repaint

weston_output_schedule_repaint实现如下：

    WL_EXPORT void
    weston_output_schedule_repaint(struct weston_output *output)
    {
      ...
      //将idle_repaint加入idle loop事件中，当服务端没有其它epoll事件时，会执行idle_repaint
    	wl_event_loop_add_idle(loop, idle_repaint, output);
      ...
    }

idle_repaints实现如下：

    static void
    idle_repaint(void *data)
    {
    	struct weston_output *output = data;
      //调用在前面wayland_output_create初始化的接口，由于默认使用drm_output_start_repaint_loop
    	output->start_repaint_loop(output);
    }

drm_output_start_repaint_loop之后的主要流程如下：

    drm_output_start_repaint_loop->
      weston_output_finish_frame->
        output_repaint_timer_handler->
          weston_output_repaint->
          output->repaint()-> // 这里的repaint为之前output初始化时初始化的drm_output_repaint
            drm_output_repaint

drm_output_repaint利用modsetting接口将绘制的内容最终显示到屏幕上，其机制还比较复杂，主要利用了pageflip、vblank和plane，相关原理与drm的API编程强相关，内容还比较多，这里就暂时不深入了，以后抽空单独写相关的文章来说明。
