---
layout: post
title:  "Wayland源码分析-damage相关流程"
date:   2016-11-10 10:52:45
author: HappySeeker
categories: Graphic
---

wayland代码分析系列，刚刚开始，慢慢来～

本文关注damage相关的流程

# Damage？

什么是damage？做图形开发的同学应该还比较熟悉，准确定义就不去深究了。

可以理解为，当图形应用需要重绘指定区域时，发送的一种事件，X11协议中有针对Damage的专门的扩展协议，Wayland中，其实就是client向server发送的一种事件(request)，server端(compositor)收到事件后进行相应的处理，通常也就是重绘指定的区域。

初始情况下，wayland客户端会将整个surface作为初始区域，发送damage，目的时绘制surface。

# 流程

还是以weston代码中的最简单的client示例代码simple-shm为例看看相关流程。

client在创建display、window之后，调用：wl_surface_damage发送request，流程开始：

wl_surface_damage实现如下(由scanner工具生成，源代码中没有)：

    static inline void
    wl_surface_damage(struct wl_surface *wl_surface, int32_t x, int32_t y, int32_t width, int32_t height)
    {
    	wl_proxy_marshal((struct wl_proxy *) wl_surface,
    			 WL_SURFACE_DAMAGE, x, y, width, height);
    }

本质上就是封装相关数据，然后调用与服务端通信的接口，将相应的request：WL_SURFACE_DAMAGE发送给server，然后由server调用本地相应的接口完成处理。

服务端接收到相应request之后，调用本地接口(有关wayland客户端和服务端通信相关的机制，后续再抽空写单独的文章来说明)，本地接口定义为：

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

可见damage对应接口为：surface_damage，实现为：

    static void
    surface_damage(struct wl_client *client,
    	       struct wl_resource *resource,
    	       int32_t x, int32_t y, int32_t width, int32_t height)
    {
    	struct weston_surface *surface = wl_resource_get_user_data(resource);

    	pixman_region32_union_rect(&surface->pending.damage,
    				   &surface->pending.damage,
    				   x, y, width, height);
    }

其实，就啥也没干，获取surface后，将新的damage区域与原有的damage区域进行组合，得到新的damage区域。

另，wayland 0.99版本之后，都使用了double buffer state，对于damage区域也是如此，之前更新damage区域时，更新的都是pending的damage，在commit之后，才将pending.damage赋值给current damage，然后clear掉pending.damage供下次使用。然后，在服务端repaint surface时，会清理掉current.damage供下次使用。

从整个流程上看，其实就是更新了一下damage对应的区域而已，没有其它操作，真正的绘图操作是在客户端commit后，由服务端compositor根据之前更新的damage区域来进行的，damage区域以外的区域不会被重绘。
