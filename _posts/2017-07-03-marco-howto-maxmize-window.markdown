---
layout: post
title:  "Marco：Normal窗口最大化流程分析"
date:   2017-07-03 17:50:32
author: JiangBiao
categories: Kernel
---

# 前言

分析Wine EDiary工具不能最大化的问题时，对比分析了SourceInsight3工具的最大化流程，由于SI3工具的主窗口是Normal类型窗口，其最大化、最小化等功能都是有窗口管理器提供的，比较典型，本文进行整体了分析，供参考。

# 分析

## 原理分析

如前一篇文章所述，Normal类型的窗口程序创建并map后，会由窗口管理器在其周围加上一个Frame，这个Frame中会包含最大化、最小化和关闭按钮，也就是说，这种类型的窗口的最大化、最小化、关闭(以及窗口移动)等功能，都是窗口管理器负责实现的，应用程序自身无需实现，这样也为应用开发者节省了工作量，降低了开发难度。

Normal类型窗口的最大化功能都是由窗口管理器负责，因为相应的按钮都位于窗口管理器提供的Frame窗口中。从整体流程上看，从鼠标点击到窗口最大化的大致过程应该是这样的：

1. 鼠标点击最大化按钮(该按钮是X11窗口中的一个控件)，会因此触发相应的X11 event。
2. Xserver会将相应的event转发到对应的X11窗口。
3. 进入gtk+事件循环，gtk+进行event分发，将其分发到指定的窗口(最大化按钮所在的窗口，位于窗口管理器创建的Frame中)。
4. 最大化按钮窗口初始化时注册的相应处理函数处理相应的event，其中会将应用程序主窗口的size设置为适当(考虑Frame)大小，并进行界面刷新等操作。这个操作在窗口管理器中完成。

## 代码分析

对于Marco窗口管理器(Mate桌面默认的窗口管理器，是Gnome桌面自带的Nautilus窗口管理器的一个分支)来说，其主要代码流程分两个部分：

1. 按钮点击事件的同步处理。
2. 空闲时resize流程。

### 按钮点击事件处理

鼠标点击最大化按钮后，最终会进入到Marco事先注册的处理回调中，这个流程不会直接设置(修改)X11窗口的大小，而只是设置了相应的标记，待idle时使用，主要流程如下

    meta_frames_button_release_event                    
      meta_core_maximize
        meta_window_maximize
          meta_window_maximize_internal                    

meta_window_maximize_internal                    中，未直接修改窗口大小，而是设置了两个关键标记：

- window->maximize_horizontally
- window->maximize_vertically

顾名思义，即在需要时进行最大化。具体代码如下：

    void
    meta_window_maximize_internal (MetaWindow        *window,
                               MetaMaximizeFlags  directions,
                               MetaRectangle     *saved_rect)
    {
      /* At least one of the two directions ought to be set */
      gboolean maximize_horizontally, maximize_vertically;
      // 传入的directions参数中包含META_MAXIMIZE_HORIZONTAL和META_MAXIMIZE_VERTICAL标记
      maximize_horizontally = directions & META_MAXIMIZE_HORIZONTAL;
      maximize_vertically   = directions & META_MAXIMIZE_VERTICAL;
      ...
      //设置标记
      window->maximized_horizontally =
      window->maximized_horizontally || maximize_horizontally;
      window->maximized_vertically =
      window->maximized_vertically   || maximize_vertically;                                              
      ...
    }

### 空闲时resize

上述流程是按钮点击后的同步处理过程，其中并未直接设置X11窗口大小，真正设置大小的操作是在空闲时进行的。在Marco中，当X11进行map操作时，由于Marco接管了所有窗口的(除override_redirect的窗口外)的事件，相应的map Request会由XServer转发到Marco中，其中会根据X11 window创建相应的Meta window，并将相应窗口入队，其中会设置window_queue_idle_handler                    ，其中就会包括idle_move_resize                    操作。整体代码流程如下：

    event_callback                    
      meta_window_new                     // event类型为MapRequest                    时
        meta_window_new_with_attrs                    
          meta_window_queue                    

meta_window_queue                    会注册idle_move_resize钩子，具体代码如下：

    void
    meta_window_queue (MetaWindow *window, guint queuebits)
    {
      guint queuenum;

      for (queuenum=0; queuenum<NUMBER_OF_QUEUES; queuenum++)
        {
          if (queuebits & 1<<queuenum)
            {
              /* Data which varies between queues.
               * Yes, these do look a lot like associative arrays:
               * I seem to be turning into a Perl programmer.
               */

              const gint window_queue_idle_priority[NUMBER_OF_QUEUES] =
                {
                  G_PRIORITY_DEFAULT_IDLE,  /* CALC_SHOWING */
                  META_PRIORITY_RESIZE,     /* MOVE_RESIZE */
                  G_PRIORITY_DEFAULT_IDLE   /* UPDATE_ICON */
                };

              const GSourceFunc window_queue_idle_handler[NUMBER_OF_QUEUES] =
                {
                  idle_calc_showing,
                  idle_move_resize,
                  idle_update_icon,
                };
                                                ...
    }

window_queue_idle_handler会由Marco在空闲时调用，idle_move_resize的主要流程如下：

    idle_move_resize
      meta_window_move_resize_now    
        meta_window_move_resize
          meta_window_move_resize_internal
            meta_window_constrain
              do_all_constraints
                constrain_maximization //其中判断window->maximized_horizontally和window->maximized_vertically标志，设置最终window的size
          XConfigureWindow                     //根据constrain_maximization设置的size，使用Xlib接口设置X11窗口大小。

具体代码就不贴了，可以自己对照翻翻。
