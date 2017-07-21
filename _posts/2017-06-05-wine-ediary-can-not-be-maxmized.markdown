---
layout: post
title:  "Wine相关问题：TOOLWINDOW类型窗口无法最大化"
date:   2017-06-05 17:40:11
author: JiangBiao
categories: Kernel
---

# 问题

使用Wine适配一个简单的笔记软件，该笔记软件的主窗口类型为TOOLWINDOW

> 无法最大化，点击按钮后闪烁几下后，只是将窗口移到了桌面的左上角，但是大小没变。

对应在窗口管理器marco中的类型为`META_WINDOW_DIALOG`

# 分析

## 原理分析

分析类似问题需要同时理解Windows窗口和X11窗口的相关机制和原理。这里仅针对**窗口最大化**这个过程进行分析。

由于wine实现本地窗口使用GTK+ ToolKit，所以对于Wine窗口程序来说，其基于GTK+现有机制实现**窗口最大化**，与普通GTK程序不同的是，wine需要处理Windows窗口相关的信息，可以将wine看作一个管道，一端是Windows窗口相关的机制；另一端是X11相关的机制。wine需要对管道两端的信息进行翻译、传递，对于wine来说，即需要理解Windows，也需要理解X11。

这里分两种情况：

1. Normal类型Wine窗口程序。这种程序最常见，比如SourceInsight3程序，其主窗口即为Normal类型。该类型窗口创建并map后，会由窗口管理器在其周围加上一个Frame，这个Frame中会包含最大化、最小化和关闭按钮，也就是说，这种类型的窗口的最大化、最小化、关闭(以及窗口移动)等功能，都是窗口管理器负责实现的，应用程序自身无需实现，这样也为应用开发者节省了工作量，降低了开发难度。同时，从职责上讲，类似的功能确实应由窗口管理器提供，职责分离，如此设计也非常合理。

2. TOOLWINDOW类型Wine窗口程序。对于这种程序的窗口，窗口管理器不负责**装饰**，即不会为其创建Frame，所以，如果这样的窗口需要实现最大化、最小化、关闭(以及窗口移动)等功能，则必须由应用程序自己实现。显然，基本没必要这样做，除非你需要非常特别的**最大化功能**，而EDiary却真这样做了。

### Normal类型Wine窗口程序

这种类型窗口的最大化功能都是由窗口管理器负责，因为相应的按钮都位于窗口管理器提供的Frame窗口中。

对于这样的程序，从整体流程上看，从鼠标点击到窗口最大化的大致过程应该是这样的：

1. 鼠标点击最大化按钮(该按钮是X11窗口中的一个控件)，会因此触发相应的X11 event。
2. Xserver会将相应的event转发到对应的X11窗口。
3. 进入gtk+事件循环，gtk+进行event分发，将其分发到指定的窗口(最大化按钮所在的窗口，位于窗口管理器创建的Frame中)。
4. 最大化按钮窗口初始化时注册的相应处理函数处理相应的event，其中会将应用程序主窗口的size设置为适当(考虑Frame)大小，并进行界面刷新等操作。这个操作在窗口管理器中完成。

窗口管理器中的具体实现，不同类型的窗口管理器的实现方式有较大差别，后面会针对marco窗口管理器的相关流程进行分析。有关窗口管理器，又是个复杂的话题，后续有时间单独写相应的文章说明。

### TOOLWINDOW类型Wine窗口程序

这种类型窗口的最大化功能由程序自己实现，不同程序的具体实现可能不同，但大致思路基本如此：

1. 在创建窗口时，在窗口的右上角创建**最大化按钮**，并为此注册单击事件处理接口。
2. 鼠标点击最大化按钮，因此触发相应的X11 event。
3. Xserver会将相应的event转发到对应的X11窗口。
4. 进入gtk+事件循环，gtk+进行event分发，将其分发到指定的窗口。
5. 调用事先注册的单击事件处理接口进行处理，其中将应用程序主窗口的size设置为适当大小，并进行界面刷新等操作。这个操作在应用程序中完成。

## 堆栈分析

由于EDiary选择了自己实现最大化功能(TOOLWINDOW类型窗口)，那其必须自己处理相关逻辑，具体如何实现，可参考如下其运行时打印的堆栈

    (gdb) bt
    #0  X11DRV_WindowPosChanged (hwnd=0x3109e, insert_after=0x0, swp_flags=6167,
        rectWindow=0x33e290, rectClient=0x33e2a0, visible_rect=0x33e170,
        valid_rects=0x0, surface=0x1eb0b0) at window.c:1295
    #1  0x7fb89b21 in set_window_pos (hwnd=0x3109e, insert_after=0x0,
        swp_flags=6167, window_rect=0x33e290, client_rect=0x33e2a0,
        valid_rects=0x0) at winpos.c:2181
    #2  0x7fb8c5bb in USER_SetWindowPos (winpos=0x33e394) at winpos.c:2254
    #3  0x7fb89dde in SetWindowPos (hwnd=0x3109e, hwndInsertAfter=0x0, x=0, y=0,
        cx=1366, cy=730, flags=22) at winpos.c:2340
    #4  0x51059482 in ?? ()
    #5  0x5106bfbf in ?? ()
    #6  0x51055c83 in ?? ()
    #7  0x510b27ae in ?? ()
    #8  0x51056ef8 in ?? ()
    #9  0x51056b73 in ?? ()
    #10 0x51026296 in ?? ()
    #11 0x7fb8e77a in WINPROC_wrapper () from /usr/bin/../lib/wine/user32.dll.so
    #12 0x7fb8edba in call_window_proc (hwnd=0x3109e, msg=5, wp=2, lp=47842646,
        result=0x33ec18, arg=0xa70f7a) at winproc.c:245
    #13 0x7fb8f21e in WINPROC_CallProcWtoA (
        callback=0x7fb8ed70 <call_window_proc>, hwnd=0x3109e, msg=5, wParam=2,
        lParam=47842646, result=0x33ec18, arg=0xa70f7a) at winproc.c:717
    #14 0x7fb91137 in WINPROC_call_window (hwnd=0x3109e, msg=5, wParam=2,
    ---Type <return> to continue, or q <return> to quit---
        lParam=47842646, result=0x33ec18, unicode=1, mapping=2142800900)
        at winproc.c:907
    #15 0x7fb552e4 in call_window_proc (hwnd=0x3109e, msg=5, wparam=2,
        lparam=47842646, unicode=1, same_thread=1, mapping=2142800900)
        at message.c:2224
    #16 0x7fb5c2ee in send_message (info=0x33ecd4, res_ptr=0x33ecd0, unicode=1)
        at message.c:3266
    #17 0x7fb5c541 in SendMessageW (hwnd=0x3109e, msg=5, wparam=2, lparam=47842646)
        at message.c:3466
    #18 0x7fb25438 in DEFWND_DefWinProc (hwnd=0x3109e, msg=<optimized out>,
        wParam=0, lParam=3406100) at defwnd.c:75
    #19 0x7fb2681a in DefWindowProcA (hwnd=<optimized out>, msg=71,
        wParam=<optimized out>, lParam=3406100) at defwnd.c:872
    #20 0x7fb8e77a in WINPROC_wrapper () from /usr/bin/../lib/wine/user32.dll.so
    #21 0x7fb8edba in call_window_proc (hwnd=0x3109e, msg=71, wp=0, lp=3406100,
        result=0x33eedc, arg=0x51007918) at winproc.c:245
    #22 0x7fb91268 in CallWindowProcA (func=<optimized out>, hwnd=<optimized out>,
        msg=<optimized out>, wParam=<optimized out>, lParam=<optimized out>)
        at winproc.c:964
    #23 0x51056fdc in ?? ()
    #24 0x51056ef8 in ?? ()
    #25 0x51056b73 in ?? ()
    #26 0x51026296 in ?? ()
    ---Type <return> to continue, or q <return> to quit---
    #27 0x7fb8e77a in WINPROC_wrapper () from /usr/bin/../lib/wine/user32.dll.so
    #28 0x7fb8edba in call_window_proc (hwnd=0x3109e, msg=71, wp=0, lp=3406100,
        result=0x33f6c8, arg=0xa70f7a) at winproc.c:245
    #29 0x7fb8f21e in WINPROC_CallProcWtoA (
        callback=0x7fb8ed70 <call_window_proc>, hwnd=0x3109e, msg=71, wParam=0,
        lParam=3406100, result=0x33f6c8, arg=0xa70f7a) at winproc.c:717
    #30 0x7fb91137 in WINPROC_call_window (hwnd=0x3109e, msg=71, wParam=0,
        lParam=3406100, result=0x33f6c8, unicode=1, mapping=3405752)
        at winproc.c:907
    #31 0x7fb552e4 in call_window_proc (hwnd=0x3109e, msg=71, wparam=0,
        lparam=3406100, unicode=1, same_thread=1, mapping=3405752)
        at message.c:2224
    #32 0x7fb5c2ee in send_message (info=0x33f784, res_ptr=0x33f780, unicode=1)
        at message.c:3266
    #33 0x7fb5c541 in SendMessageW (hwnd=0x3109e, msg=71, wparam=0, lparam=3406100)
        at message.c:3466
    #34 0x7fb8c72f in USER_SetWindowPos (winpos=0x33f914) at winpos.c:2308
    #35 0x7fb89dde in SetWindowPos (hwnd=0x3109e, hwndInsertAfter=0x0, x=0, y=0,
        cx=1366, cy=730, flags=32800) at winpos.c:2340
    #36 0x7fb8b5bc in show_window (hwnd=0x3109e, cmd=3) at winpos.c:1160
    #37 0x7fb8b8a8 in ShowWindow (hwnd=0x3109e, cmd=3) at winpos.c:1260
    #38 0x5106e998 in ?? ()
    #39 0x51056ef8 in ?? ()
    ---Type <return> to continue, or q <return> to quit---
    #40 0x51026296 in ?? ()
    #41 0x7fb8e77a in WINPROC_wrapper () from /usr/bin/../lib/wine/user32.dll.so
    #42 0x7fb8edba in call_window_proc (hwnd=0x210dc, msg=514, wp=0, lp=917512,
        result=0x33fcfc, arg=0xa70f53) at winproc.c:245
    #43 0x7fb910d5 in WINPROC_call_window (hwnd=0x210dc, msg=514, wParam=0,
        lParam=917512, result=0x33fcfc, unicode=0,
        mapping=WMCHAR_MAP_DISPATCHMESSAGE) at winproc.c:920
    #44 0x7fb571fb in DispatchMessageA (msg=0x33fdec) at message.c:3961
    #45 0x51074154 in ?? ()
    #46 0x511989b6 in ?? ()

堆栈有点长，只需关注几个关键点：

1. \#37堆栈，ShowWindow的参数中cmd=3，查看MSDN即可了解cmd=3表示，要执行Maxmize操作。Windows中，ShowWindow是一个很常用的用户编程接口，在通常在创建窗口后，如果要显示该窗口，就需要调用该函数(根据传入cmd参数不同，作用不同，当cmd=1时，作用跟Xlib中的XMapWindow()类似)，仅在调用后，相应窗口才Visible，才能显示。而ShowWindow除**显示窗口**外，还有很多作用，比如隐藏、最大化、最小化窗口，具体见MSDN。下面摘取了几个典型的：

    SW_HIDE  0  Hides the window and activates another window.

    SW_MAXIMIZE  3  Maximizes the specified window.

    SW_MINIMIZE  6  Minimizes the specified window and activates the next top-level window in the Z order.

    SW_SHOWNORMAL  1  Activates and displays a window.

2. \#35堆栈，设置窗口大小，传入的参数中，cx和cy与当前的分辨率对应，以此达到最大化的目的。

3. \#33堆栈，在USER_SetWindowPos函数中会向窗口发送编号为71的message，查看MSDN，可以确认71对应的message为WM_WINDOWPOSCHANGED。显然，应用程序中需要处理相应消息，否则会调用系统默认的处理流程。

4. \#22-\#18堆栈，说明应用程序没有注册WM_WINDOWPOSCHANGED消息的回调，则进入wine定义的默认流程处理：DEFWND_DefWinProc。

5. \#17堆栈，DEFWND_DefWinProc中会向窗口发送WM_SIZE消息(根据MSDN，表明窗口大小发生改变)，应用程序中应该注册相应回调处理该消息，以实现**最大化**，EDiary的确是这样做的。

6. \#3堆栈，EDiary中注册了对WM_SIZE消息的处理，其中调用SetWindowPos，设置窗口大小为屏幕大小，以实现窗口最大化。

从这个堆栈看，流程比较清晰，看似没有问题，但最终却没有最大化成功。

## 代码分析

最大化对应的本质操作为：设置窗口大小，只是将窗口大小设置为屏幕大小。

从上述的堆栈看，EDiary实现最大化是通过调用SetWindowPos接口，该接口用于设置窗口的位置、大小、z序等属性，使用频率非常高，详见MSDN。

在wine的具体实现中，SetWindowPos接口实现比较复杂，其中，设置窗口大小的主要代码流程为：
>	
>	SetWindowPos
>	  SER_SetWindowPos
>	    set_window_pos
>	      X11DRV_WindowPosChanged
>	        sync_window_position
>	          XReconfigureWMWindow
>	            XConfigureWindow

该流程最终调用Xlib接口XConfigureWindow设置窗口大小，看似也没有问题。

### 问题点1

通过gdb跟踪相关流程后，发现EDiary在点击最大化按钮对应的事件处理流程中，没有进入sync_window_position流程。

看看相关代码：

        /* don't change position if we are about to minimize or maximize a managed window */
        if (!event_type &&
            !(data->managed && (swp_flags & SWP_STATECHANGED) && (new_style & (WS_MINIMIZE|WS_MAXIMIZE))))
            sync_window_position( data, swp_flags, &old_window_rect, &old_whole_rect, &old_client_rect );
         
可以看出，对应new_style为WS_MAXIMIZE的窗口来说，其不调用sync_window_position，注释中也有说明，原因大家也能想明白，看似最大化的窗口不能再设置大小，看起来没错，但对于像EDiary这样的程序来说，即在点击最大化按钮后，其new_style显然为WS_MAXIMIZE了，但是，其事实上并未出于最大化状态，相反，其恰恰依赖于这个流程来实现最大化，所以，这里的逻辑看似有点问题，需要优化，至少对于EDiary来说需要。

### 问题点2

再进入sync_window_position函数，看看里面的具体实现：

    /***********************************************************************
     *		sync_window_position
     *
     * Synchronize the X window position with the Windows one
     */
    static void sync_window_position( struct x11drv_win_data *data,
                                      UINT swp_flags, const RECT *old_window_rect,
                                      const RECT *old_whole_rect, const RECT *old_client_rect )
    {
        // 获取窗口属性
        DWORD style = GetWindowLongW( data->hwnd, GWL_STYLE );
        DWORD ex_style = GetWindowLongW( data->hwnd, GWL_EXSTYLE );
        XWindowChanges changes;
        unsigned int mask = 0;

        if (data->managed && data->iconic) return;

        /* resizing a managed maximized window is not allowed */
        // 这里的逻辑也有类似的问题，对于style为WS_MAXIMIZE，不设置mask |= CWWidth | CWHeight;即不修改窗口大小
        if (!(style & WS_MAXIMIZE) || !data->managed)
        {
            changes.width = data->whole_rect.right - data->whole_rect.left;
            changes.height = data->whole_rect.bottom - data->whole_rect.top;
            /* if window rect is empty force size to 1x1 */
            if (changes.width <= 0 || changes.height <= 0) changes.width = changes.height = 1;
            if (changes.width > 65535) changes.width = 65535;
            if (changes.height > 65535) changes.height = 65535;
            mask |= CWWidth | CWHeight;
        }
        // 设置窗口起始位置对应的mask
        /* only the size is allowed to change for the desktop window */
        if (data->whole_window != root_window)
        {
            POINT pt = virtual_screen_to_root( data->whole_rect.left, data->whole_rect.top );
            changes.x = pt.x;
            changes.y = pt.y;
            mask |= CWX | CWY;
        }
        // 设置Z序调整对应的mask
        if (!(swp_flags & SWP_NOZORDER) || (swp_flags & SWP_SHOWWINDOW))
        {
            /* find window that this one must be after */
            HWND prev = GetWindow( data->hwnd, GW_HWNDPREV );
            while (prev && !(GetWindowLongW( prev, GWL_STYLE ) & WS_VISIBLE))
                prev = GetWindow( prev, GW_HWNDPREV );
            if (!prev)  /* top child */
            {
                changes.stack_mode = Above;
                mask |= CWStackMode;
            }
            /* should use stack_mode Below but most window managers don't get it right */
            /* and Above with a sibling doesn't work so well either, so we ignore it */
        }

        set_size_hints( data, style );
        set_mwm_hints( data, style, ex_style );
        data->configure_serial = NextRequest( data->display );
        // 调用XConfigureWindow实现X11窗口属性的修改
        XReconfigureWMWindow( data->display, data->whole_window, data->vis.screen, mask, &changes );
    #ifdef HAVE_LIBXSHAPE
        if (IsRectEmpty( old_window_rect ) != IsRectEmpty( &data->window_rect ))
            sync_window_region( data, (HRGN)1 );
        if (data->shaped)
        {
            int old_x_offset = old_window_rect->left - old_whole_rect->left;
            int old_y_offset = old_window_rect->top - old_whole_rect->top;
            int new_x_offset = data->window_rect.left - data->whole_rect.left;
            int new_y_offset = data->window_rect.top - data->whole_rect.top;
            if (old_x_offset != new_x_offset || old_y_offset != new_y_offset)
                XShapeOffsetShape( data->display, data->whole_window, ShapeBounding,
                                   new_x_offset - old_x_offset, new_y_offset - old_y_offset );
        }
    #endif

        TRACE( "win %p/%lx pos %d,%d,%dx%d after %lx changes=%x serial=%lu\n",
               data->hwnd, data->whole_window, data->whole_rect.left, data->whole_rect.top,
               data->whole_rect.right - data->whole_rect.left,
               data->whole_rect.bottom - data->whole_rect.top,
               changes.sibling, mask, data->configure_serial );
    }