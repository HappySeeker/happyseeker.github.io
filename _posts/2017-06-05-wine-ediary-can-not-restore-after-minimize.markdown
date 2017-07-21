---
layout: post
title:  "Wine ediary在最小化后无法恢复问题"
date:   2017-07-04 09:48:18
author: HappySeeker
categories: Kernel
---

# 问题

ediary是个很简单的Windows笔记软件，相对小众，但易用性还不错，还是有不少fans。

尝试使用Wine对其进行适配时，发现其基本功能可用，但有一个影响使用的大问题：

在最小化后，其会进入托盘区（systray），点击后会正常弹出密码输入提示框，但输入密码后，没有反应，看不到本应出现的笔记编辑窗口。

# 分析

## 从现象出发

问题现象本质为**窗口未显示**，可能是窗口没有执行map操作。可以通过如下命令列举当前所有的窗口：

    xwininfo -root -tree

从中根据窗口的name和大小等信息确认EDiary对应的程序主窗口，获取其id。

然后，命令尝试用自己编写的程序，根据上面获取的id，对其强制调用XMapWindow并Raise其窗口位置。结果窗口显示出来了，说明问题确实在于未对该窗口进行适当的map的操作，看似问题明了了，但具体实质问题根源还相距甚远。

## 确认关键点

Wine作为Windows窗口机制和X11之前的桥梁，需要负责Windows到X11的接口翻译工作，与X11中XMapWindow接口对应的Windows操作接口为：ShowWindow，该接口之前介绍过，详见MSDN，其作用不止于显示窗口、也能隐藏、最小化、最大化窗口。

对于某Windows窗口，要使其显示，绝大部分情况都会调用ShowWindow接口，所以，在分析时，需要在ShowWindow接口处打点。

另外，通常Windows图形应用都会有比较复杂的窗口层次关系，比如创建的程序主窗口并非实际显示的窗口，实际显示窗口可能仅是程序主窗口的子窗口，而所有窗口创建都会调用CreateWindowEX接口，所以，gdb跟踪时，最好也在这个地方打上点。

## Wine EDiary窗口显示相关流程

通过gdb跟踪，可以看到通过Wine模拟的EDiary工具的显示和与用户交互的大致流程如下

1. 创建程序主窗口。该窗口大小为1x1，并不直接用于显示，而是作为所有窗口的父窗口。
2. 创建并显示登录界面窗口。该窗口用于输入密码，是程序主窗口的子窗口。
3. 用户输入密码，并点击确认后，创建并显示程序主界面窗口，该窗口是用户编辑和使用的主要窗口。
4. 用户点击最小化按钮(或者操作其它窗口，EDiary工具设置有定时器，一段时间不操作后，会自动最小化)，使用ShowWindow接口Hide主界面窗口，并在托盘区创建相应的图标。相应应用不在panel中显示。
5. 用户点击托盘区图标，弹出登录界面窗口，供用户输入密码。至此，所有操作正常，与Windows中的表现无异。
6. 用户输入密码后，Hide登录界面窗口，但主界面窗口并未正常显示。

从流程上看，关键问题在于，上述第6步中，用户在输入密码后，主界面窗口并未正常显示，没有跟踪到相应的ShowWindow操作。

问题看似简单，是否就在相应的代码中，对指定窗口加一个ShowWindow操作就可以了？

显然，这也许能规避问题，但并不规范，并未找到根源所在。

## 关键流程跟踪

使用gdb重点对ShowWindow相关流程进行跟踪，从点击托盘区图标开始，跟踪到的关键流程如下：

1. 创建输密码的窗口
2. showwindow(APP主窗口，5) //5表示SW_SHOW(详见MSDN)，systray窗口msg=45416
3. 创建输密码的相关子窗口
4. showwindow(输密码的窗口，1) //1表示SW_SHOWNORMAL
5. showwindow(APP主窗口，9) // 9表示SW_Restore(详见MSDN)，此时，输密码窗口显示(还没输入密码)，这里可能有时序问题？
6. showwindow(APP主窗口，5) // 5表示SW_SHOW

这个流程中有几个关键点：

1. 主界面窗口并没有调用showwindow操作。
2. 上述第5步中的Restore操作的出现有点奇怪，需要确认来源。

## 对比Windows关键流程

问题到这里，似乎很难继续，因为**对窗口进行Show**这个关键操作应该是由EDiary程序自己发起的，不应该在wine中进行，所以，如果EDiary没有调用相应操作(ShowWindow)，那实际上wine也没有办法，不能控制。那么为什么EDiary没有对程序主界面窗口调用ShowWindow操作呢？目前不清楚，因为EDiary闭源，看不到代码，其内部实现黑盒，类似的问题很难定位。

但其实还有其它办法可以继续分析。Windows中有很多的Hack工具，可以跟踪Windows程序的调用的系统接口(包括Windows提供的各种系统DLL中提供的接口)。通过这样的工具可以分析EDiary工具在Windows中的关键函数调用过程，再与Wine环境中做对比(Wine中可以的通过类似relay的调试开关看到Windows应用调用的所有Windows接口，很多～～，需要花很大的精力来梳理)，即可看出EDiary在Windows和Wine的运行流程上的差异。

在网上找了个ApiSpy工具，口碑不错，自测感觉也很好用。在Windows中安装运行此程序，并在其中启动EDiary工具，对其进行监控，发现在Windows(Win7)环境中，其关键流程如下(从点击systray中的图标开始)：

1. 创建输密码的窗口
2. showwindow(APP主窗口，5)
3. 创建输密码的相关子窗口
4. showwindow(输密码的窗口，1)
5. 输密码窗口显示，输入密码，点击确认后
6. showwindow(APP主窗口，9) 其中会showwindow(主界面窗口， shownoactive)
7. showwindow(APP主窗口，5)

可以看出，整体流程看似跟wine差不多，但有3个关键区别：

1. **在输入密码并点击确认后**才进行第6步中的Restore操作(cmd=9)。
2. 第6步中的Restore操作是EDiary程序直接调用的，而Wine中是由其它流程触发的。
3. 在第6步中，在Restore程序主窗口的过程中，会对程序主界面窗口调用ShowWindow操作。

## 流程对比分析

### Restore与owned window

先看第3点，这点最关键。

对于EDiary来说，程序主窗口(大小为1x1,为所有窗口的父窗口)和程序主界面窗口(大小为桌面大小，是用户主要交互窗口，问题所在)，之间是owner和owned关系，即程序主窗口拥有程序主界面窗口，有关Windows中owner和owned窗口的具体定义和关系请参考MSDN。

从Windows系统中EDiary的调用流程看，在owner窗口的Restore过程中，会触发owned窗口的Show操作。查看MSDN，可以确认这是有官方依据的，MSDN相关原文：

>When an owner window is minimized, the system automatically hides the associated owned windows. Similarly, when an owner window is restored, the system automatically shows the associated owned windows. In both cases, the system sends the WM_SHOWWINDOW message to the owned windows before hiding or showing them. Occasionally, an application may need to hide the owned windows without having to minimize or hide the owner. In this case, the application uses the ShowOwnedPopups function. This function sets or removes the WS_VISIBLE style for all owned windows and sends the WM_SHOWWINDOW message to the owned windows before hiding or showing them. Hiding an owner window has no effect on the visibility state of the owned windows.


相关参考链接：

[MSDN描述] <https://msdn.microsoft.com/en-us/library/windows/desktop/ms632599(v=vs.85).aspx>

大致意思就是说，当Restore owner窗口时，应该自动show owned窗口。

很明显，Windows中流程与MSDN保持一致，而Wine中并没有这样做。Wine中相关代码流程：

    ShowWindow
      show_window
        SetWindowPos
          USER_SetWindowPos
            set_window_pos

在show_window函数中对Restore操作进行了判断，但并未做任何处理：

    /***********************************************************************
     *              show_window
     *
     * Implementation of ShowWindow and ShowWindowAsync.
     */
    static BOOL show_window( HWND hwnd, INT cmd )
    {
        ...

        switch(cmd)
        {
            case SW_HIDE:
                ...
            case SW_RESTORE:
            /* fall through */
	          case SW_SHOWNORMAL:  /* same as SW_NORMAL: */
	          case SW_SHOWDEFAULT: /* FIXME: should have its own handler */
            ...
        }
      }

看似，wine并未遵守MSDN的约定，但其实也不完全，在另一个流程中(处理SYSCOMMAND的流程中，通常图形控件上的鼠标操作会走这个流程)，其实也对owned操作进行show操作，相关代码如下：

    /***********************************************************************
     *           NC_HandleSysCommand
     *
     * Handle a WM_SYSCOMMAND message. Called from DefWindowProc().
     */
    LRESULT NC_HandleSysCommand( HWND hwnd, WPARAM wParam, LPARAM lParam )
    {
    	...
      case SC_RESTORE:
          if (IsIconic(hwnd) && hwnd == GetActiveWindow())
              ShowOwnedPopups(hwnd,TRUE);
          ShowWindow( hwnd, SW_RESTORE );
          break;
        ...
      }
      
相关代码流程如下：
    DefWindowProcW  // wine定义的默认窗口函数，负责处理应用程序窗口
                    // 函数中未处理的message，被应用程序中定义的窗口函数调用。
      DEFWND_DefWinProc
        NC_HandleSysCommand
        
可以看出，wine也应该了解msdn中的相关约定，但遵守不完全，忽略了应用程序直接调用ShowWindow(Restore)接口的情况，这种情况下，owned窗口不会被显示。所以，这里遇到的问题的主要原因于此。

### Restore的来源

针对上述疑点，在修改show_window中的代码后，发现问题仍未解决，还需继续分析，另一个关键疑点：wine中跟踪的关键流程第5步中，Restore操作的来源，为何会在还未输入密码之前出现？

使用gdb跟踪，打印其堆栈如下：

    (gdb) bt
    #0  ShowWindow (hwnd=0xb06f6, cmd=9) at winpos.c:1239
    #1  0x7bc7b86e in relay_call () from /lib/wine/ntdll.dll.so
    #2  0x7fa69f15 in __wine_spec_relay_entry_points ()
       from /lib/wine/user32.dll.so
    #3  0x51072c9c in ?? ()
    #4  0x51026296 in ?? ()
    #5  0x7fb09dfa in WINPROC_wrapper () from /lib/wine/user32.dll.so
    #6  0x7fb0a547 in call_window_proc (hwnd=0xb06f6, msg=274, wp=61728, lp=0,
        result=0x33f554, arg=0x960fef) at winproc.c:245
    #7  0x7fb0aaa1 in WINPROC_CallProcWtoA (callback=<optimized out>,
        hwnd=<optimized out>, msg=274, wParam=61728, lParam=0, result=0x33f554,
        arg=0x960fef) at winproc.c:859
    #8  0x7fb0cf37 in WINPROC_call_window (hwnd=0xb06f6, msg=274, wParam=61728,
        lParam=0, result=0x33f554, unicode=1, mapping=2130074987) at winproc.c:907
    #9  0x7fac5f6a in call_window_proc (hwnd=0xb06f6, msg=274, wparam=61728,
        lparam=0, unicode=1, same_thread=1, mapping=2130074987) at message.c:2224
    #10 0x7facdf3a in send_message (info=0x33f620, res_ptr=0x33f61c, unicode=1)
        at message.c:3279
    #11 0x7face216 in SendMessageW (hwnd=0xb06f6, msg=274, wparam=61728, lparam=0)
        at message.c:3479
    #12 0x7ef3d4d8 in handle_wm_state_notify (hwnd=0xb06f6, update_window=1,
        event=<optimized out>, event=<optimized out>) at event.c:1330
    #13 0x7ef3d8ce in X11DRV_PropertyNotify (hwnd=0xb06f6, xev=0x33f75c)
    #14 0x7ef3e707 in process_events (display=<optimized out>,
        arg=<optimized out>, filter=<optimized out>) at event.c:399
    #15 0x7ef3ef97 in X11DRV_MsgWaitForMultipleObjectsEx (count=0, handles=0x0,
        timeout=0, mask=1279, flags=0) at event.c:494
    #16 0x7bc7b86e in relay_call () from /lib/wine/ntdll.dll.so
    #17 0x7ef2e9d5 in __wine_spec_relay_entry_points ()
       from /lib/wine/winex11.drv.so
    #18 0x7fb0a880 in wait_message (count=0, handles=0x0, timeout=0, mask=1279,
        flags=0) at winproc.c:1136
    #19 0x7facd0ae in PeekMessageW (msg_out=<optimized out>, hwnd=<optimized out>,
        first=<optimized out>, last=<optimized out>, flags=<optimized out>)
        at message.c:3788
    #20 0x7facd31e in PeekMessageA (msg=0x33fa64, hwnd=0x0, first=0, last=0,
        flags=1) at message.c:3815
    #21 0x7bc7b86e in relay_call () from /lib/wine/ntdll.dll.so
    #22 0x7fa691ad in __wine_spec_relay_entry_points ()
       from /lib/wine/user32.dll.so
    #23 0x510740e4 in ?? ()
    #24 0x51172dbe in ?? ()
    #25 0x510c4a94 in ?? ()
    #26 0x510c4d71 in ?? ()
    #27 0x51026296 in ?? ()
    #28 0x7fb09dfa in WINPROC_wrapper () from /lib/wine/user32.dll.so
    #29 0x7fb0a547 in call_window_proc (hwnd=0xe06cc, msg=45156, wp=0, lp=514,
        result=0x33fc68, arg=0x960f12) at winproc.c:245
    #30 0x7fb0cecb in WINPROC_call_window (hwnd=0xe06cc, msg=45156, wParam=0,
        lParam=514, result=0x33fc68, unicode=0, mapping=WMCHAR_MAP_DISPATCHMESSAGE)
        at winproc.c:920
    #31 0x7fac8184 in DispatchMessageA (msg=0x33fd9c) at message.c:3974
    #32 0x7bc7b86e in relay_call () from /lib/wine/ntdll.dll.so
    #33 0x7fa6728d in __wine_spec_relay_entry_points ()
       from /lib/wine/user32.dll.so
    #34 0x51074154 in ?? ()
    #35 0x511989b6 in ?? ()
    #36 0x7b46af09 in call_process_entry () from /lib/wine/kernel32.dll.so

又是个长堆栈，不好看，其实熟悉后，看起来还是很容易的，该堆栈对应的简化流程为：

1. 用户点击托盘区(systray)图标
2. systray窗口产生相应Message(msg=45416)
3. wine消息循环中调用PeekMessage
4. wait_message
5. process_events
6. X11DRV_PropertyNotify
7. handle_wm_state_notify
8. SendMessageW(msg=274(SYSCOMMAND), F120(Restore)) //向窗口发送Restore消息

自然语言描述：当用户点击托盘区图标，触发相应的事件(PropertyNotify)，wine中处理PropertyNotify事件，并向窗口发送Restore的SYSCOMMAND。

如之前的描述，在这个流程中会进行owned window的show操作，但这不是关键，关键的是为何要在PropertyNotify的处理过程中发Restore消息？相关代码如下：

    /***********************************************************************
     *           handle_wm_state_notify
     *
     * Handle a PropertyNotify for WM_STATE.
     */
    static void handle_wm_state_notify( HWND hwnd, XPropertyEvent *event, BOOL update_window )
    {
        ...
        style = GetWindowLongW( data->hwnd, GWL_STYLE );
        /*
         * 这里判断当前窗口是否是iconic状态(即最小化状态)，同时wm_state(即窗口管理器测对应的窗口状态)
         * 是否是NormalState。这里需要仔细理解下，data中的iconic表示的是X11的窗口状态，而wm_state
         * 则是窗口管理器侧的。同时满足这两个条件表明X11的窗口状态已经设置为了最小化状态，但是窗口管理器
         * 中的状态，还没有变成IconicState，这种情况本身应该是一种异常，按理状态应该一致才对，出现
         * 这种情况时，对窗口进行Restore操作，看似也是合理的。而对于EDiary，则刚好满足这种条件。
         */
        if (data->iconic && data->wm_state == NormalState)  /* restore window */
        {
            data->iconic = FALSE;
            read_net_wm_states( event->display, data );
            if ((style & WS_CAPTION) == WS_CAPTION && (data->net_wm_state & (1 << NET_WM_STATE_MAXIMIZED)))
            {
                if ((style & WS_MAXIMIZEBOX) && !(style & WS_DISABLED))
                {
                    TRACE( "restoring to max %p/%lx\n", data->hwnd, data->whole_window );
                    release_win_data( data );
                    SendMessageW( hwnd, WM_SYSCOMMAND, SC_MAXIMIZE, 0 );
                    return;
                }
                TRACE( "not restoring to max win %p/%lx style %08x\n", data->hwnd, data->whole_window, style );
            }
            else
            {
                // 这里开始，判断当前style，如果是最小化或者最大化状态，是的话，发送Restore对应的消息。
                if (style & (WS_MINIMIZE | WS_MAXIMIZE))
                {
                    TRACE( "restoring win %p/%lx\n", data->hwnd, data->whole_window );
                    release_win_data( data );
                    SendMessageW( hwnd, WM_SYSCOMMAND, SC_RESTORE, 0 );
                    return;
                }
                TRACE( "not restoring win %p/%lx style %08x\n", data->hwnd, data->whole_window, style );
            }
        }
    	...
    }
    
如注释中的说明：代码中判断当前窗口是否是iconic状态(即最小化状态)，同时wm_state(即窗口管理器测对应的窗口状态)，是否是NormalState。同时满足这两个条件表明X11的窗口状态已经设置为了最小化状态，但是窗口管理器中的状态，还没有变IconicState，这种情况本身应该是一种异常(按理状态应该一致才对)，出现这种情况时，对窗口进行Restore操作，看似也是合理的。

而对于EDiary，则刚好满足这种条件。至于为何会满足这个条件，从整体上分析，应该是由于其为TOOLWINDOW类型窗口，其**最小化**功能是其自己实现的，没有使用窗口管理器自带的功能(Normal类型窗口都使用窗口管理器实现)，这种情况下，需要自行处理本地状态和窗口管理器中的状态同步问题，此时很容易造成状态不一致，详细流程没有时间再做深入分析了，有兴趣的同学可以自己试试。

总的来说，EDiary的最小化功能(与另一篇文章中分析的最大化功能异常一样)实现有缺陷，至少在wine平台下会表现出缺陷。建议不要自己实现类似的功能，自找没趣。

还有一个问题：这里莫名给主界面窗口发送Restore消息后，为啥会导致主界面窗口无法恢复？

这个问题应该跟EDiary的自身实现有关，由于是黑盒，具体不清楚。推断是由于未预期的Restore消息打乱了EDiary的消息节奏(有冲突)，导致在用户输入密码后不再调用ShowWindow(Restore)了，而主界面窗口正是依赖ShowWindow(Restore)来进行显示，在这种情况下，就无法再显示出来了。

## 解决

所以，导致这个问题的原因有两个，需要分别修复。

1. owner窗口Restore时未恢复owned窗口，wine实现与MSDN不一致，需要修正。
2. wine中处理PropertyNotify事件时，如果窗口本地状态与窗口管理器的状态不一致，则会向窗口发送Restore的SYSCOMMAND，会导致EDiary处理异常。这个问题解决方法很多，可以想办法保证状态一致(比较难)，也可以针对EDiary屏蔽相应的Restore消息(比较简单)。

