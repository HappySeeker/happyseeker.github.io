---
layout: post
title:  "Wayland源码分析-Commit相关流程"
date:   2016-11-10 11:51:30
author: HappySeeker
categories: Graphic
---


本文关注Wayland中从客户端执行wl_surface_commmit后相关的流程

# Commit？

为什么需要commit操作？

因为，surface state需要double buffer，duouble buffer有两个好处：

1. 防止抖动
2. 效率更高

surface state包括：input、opaque region、damage region、attached buffer等。

wayland协议要求surface state必须是双缓存的，因此在commit之前的所有对surface state的更新操作都是针对pending state的，只有在执行wl_surface_commmit之后，才将所有pending state更新到current state中。

在commit时，，所有其它状态在pending wl_buffer更新后更新，这就意味着所有的双缓存的state中的坐标都是相对于即将更新的wl_buffer的，如果没有wl_buffer，则坐标相对于current surface。

# 示例
weston代码中的最简单的client示例代码simple-shm中的redraw流程为例，看看commit操作如何使用。

    static void
    redraw(void *data, struct wl_callback *callback, uint32_t time)
    {
    	struct window *window = data;
    	struct buffer *buffer;
      //获取下一个用于绘图的buffer
    	buffer = window_next_buffer(window);
    	if (!buffer) {
    		fprintf(stderr,
    			!callback ? "Failed to create the first buffer.\n" :
    			"Both buffers busy at redraw(). Server bug?\n");
    		abort();
    	}
      //向buffer中绘制数据，直接修改像素值
    	paint_pixels(buffer->shm_data, 20, window->width, window->height, time);
      //将有绘制数据的buffer attach到当前windows的surface上，便于后续显示
    	wl_surface_attach(window->surface, buffer->buffer, 0, 0);
      //向服务端发送damage请求，实际上就是根据之前绘图操作的区域，更新相应damage区域。
      wl_surface_damage(window->surface,
    			  20, 20, window->width - 40, window->height - 40);

    	if (callback)
    		wl_callback_destroy(callback);
      //设置回调
    	window->callback = wl_surface_frame(window->surface);
    	wl_callback_add_listener(window->callback, &frame_listener, window);
      //commit操作
    	wl_surface_commit(window->surface);
    	buffer->busy = 1;
    }

从上述示例可以看出，在客户端完成绘图，并完成所有相关操作后，再进行commit操作，然后之前进行的所有操作才能体现到最终的界面中。

# 流程

还是以weston代码中的最简单的client示例代码simple-shm为例看看相关流程。
wl_surface_commit为流程的入口，其实现如下(由scanner工具生成，源代码中没有)：

    static inline void
    wl_surface_commit(struct wl_surface *wl_surface)
    {
    	wl_proxy_marshal((struct wl_proxy *) wl_surface,
    			 WL_SURFACE_COMMIT);
    }

本质上就是封装相关数据，然后调用与服务端通信的接口，将相应的request：WL_SURFACE_COMMIT发送给server，然后由server调用本地相应的接口完成处理。

服务端接收到相应request之后，调用本地接口，本地接口定义为：

    static const struct wl_surface_interface surface_interface = {
    	surface_destroy,
    	surface_attach,
    	surface_damage,
    	surface_frame,
    	surface_set_opaque_region,
    	surface_set_input_region,
    	surface_commit,
    	surface_set_buffer_transform,
    	surface_set_buffer_scale
    };

可见commit对应接口为：surface_commit，实现为：

    static void
    surface_commit(struct wl_client *client, struct wl_resource *resource)
    {
      //获取surface
    	struct weston_surface *surface = wl_resource_get_user_data(resource);
      //获取subsurface
      struct weston_subsurface *sub = weston_surface_to_subsurface(surface);

    	if (sub) {
    		weston_subsurface_commit(sub);
    		return;
    	}
      //关键接口
    	weston_surface_commit(surface);

    	wl_list_for_each(sub, &surface->subsurface_list, parent_link) {
    		if (sub->surface != surface)
    			weston_subsurface_parent_commit(sub, 0);
    	}
    }

代码很简单，主要就是调用了weston_surface_commit，继续看看：

    static void
    weston_surface_commit(struct weston_surface *surface)
    {
      //将pending state更新到current state中，state包括前面描述的诸多内容
    	weston_surface_commit_state(surface, &surface->pending);
      //更新subsurface的顺序，其实也是将subsurface从pending列表换到当前的列表中
    	weston_surface_commit_subsurface_order(surface);
      //重绘界面
    	weston_surface_schedule_repaint(surface);
    }

  还是很简单，主要在更新surface的state后，进入了repaint流程，repaint就是重新绘制界面，但只绘制damage区域中的内容，本质上就是将surface对应的buffer中的damage区域中的内容，重新拷贝到framebuffer中，但期间还需要由compositor根据需要进行合成。具体绘制过程又是另一个话题了，单独的文章中讨论。

  总体看，commit操作本质上就是为double buffer而设计，多次操作、一次提交，很多软件实现中都有类似的思想。
