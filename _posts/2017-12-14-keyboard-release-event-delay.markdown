---
layout: post
title:  "Spice中键盘事件延迟问题分析"
date:   2017-12-14 17:12:22
author: JiangBiao
categories: Kernel
---

#  故障现象

通过Spice客户端远控虚拟桌面，在Spice中的窗口中敲键盘按键，偶而会出现：连续重复按键的问题。即当你敲入一个a键，但出来一堆的a，直到你再敲入一个其它按键后，才停止显示a。
出现这种情况，常常会让人抓狂，现场的用户在通过云桌面练习打字时，都差点急哭了。

# 分析

## Big Picture

从理论上，键盘按键相应的字符从手指按下开始，到最终spice中的界面中显示出来，其对应的键盘事件需要大体经历如下的过程：

1. 键盘按键触发硬件中断
2. 内核中的ISR处理中断
3. ISR处理完成后，向用户态Xorg转发相应的事件
4. Xorg中的XKB模块处理相应事件，并根据当前焦点区域，将其转换为XEvent事件，并将其转发给相应的Client程序(这里即Spice客户端程序)
5. Spice客户端程序中监控相应事件，并进一步将其转发到Spice server.
6. Spice server收到事件后，进行处理，比如在文本编辑器中显示字符。并将显示后的结果通过网络发送给Spice客户端。

这是一个比较长且复杂的过程。理解了这个过程后，才能进一步深入。

对于一个按键操作，通常会先后触发两个事件：KeyPress和KeyRelease事件，分别对应按键的按下和弹起，普通键盘这两个事件的时间差在100ms左右。对于当前的现象，可以推断KeyPress事件肯定是收到了，否则不会有字符显示，很可能是KeyRelease事件没有收到，或者有延时，此时通常客户端程序会有一定的处理机制，会认为用户一直按下了按键没有释放，通常会一直重复显示相关字符。Xorg中也有类似的机制。

基于这个思路，就可以开始分析了。分析过程比较曲折，整个分析过程分成了几个阶段。  
- spice打点
- Xorg打点
- kernel打点
- spice再次打点
- 理论和代码分析

## 阶段1：spice打点

从现象出发，在spice中XEvent的处理流程中进行打点，确认Spice是否及时正常的收到的相应的键盘事件，通过打点确认，确认KeyRelease事件确实有很大的延迟(但没有丢)，导致这样的现象。

Spice代码中，使用select接口监控来自Xorg的XEvent，相关代码如下：
主线程的主循环代码：

ProcessLoop::run()：

	int ProcessLoop::run()
	{
	    _thread = pthread_self();
	    _started = true;
	    on_start_running();
			// 死循环处理消息
	    for (;;) {
					// 等待事件上来，这里的_event_sources是个抽象对象，具体实现可能是不能的事件源，其中包括XEvent
	        if (_event_sources.wait_events(_timers_queue.get_soonest_timeout())) {
	            _quitting = true;
	            break;
	        }
					// 执行定时器任务
	        _timers_queue.timers_action();
					// 处理事件队列，会清空队列
	        process_events_queue();
	        if (_quitting) {
	            break;
	        }
	    }

	    return _exit_code;
	}

ProcessLoop::run()->EventSources::wait_events()：

	// 在主线程的主循环中调用，用于等待(监控)事件
	bool EventSources::wait_events(int timeout_msec)
	{
	    int maxfd = 0;
	    fd_set rfds;
	    struct timeval tv;
	    struct timeval *tvp;
	    int ready;

	    FD_ZERO(&rfds);

	    int size = _events.size();
	    for (int i = 0; i < size; i++) {
				maxfd = MAX(maxfd, _fds[i]);
				FD_SET(_fds[i], &rfds);
	    }

	    if (timeout_msec == INFINITE) {
	        tvp = NULL;			
	    } else {
				tv.tv_sec = timeout_msec / 1000;
				tv.tv_usec = (timeout_msec % 1000) * 1000;
				tvp = &tv;
	    }
			/*
			 * 这里调用select监控Xorg连接对应的fd，只监控read状态，rfds用于返回可read状态的fd，
			 * 这样可以同时监控多个fd，实现多路复用，同时还可以设置超时。
			 */
	    /* Right now we only use read polling in spice */
	    ready = ::select(maxfd+1, &rfds, NULL, NULL, tvp);

	    if (ready == -1) {
				if (errno == EINTR) {
		    	return false;
				}
				THROW("wait error select failed");
	    } else if (ready == 0) {
				return false;
	    }

	    for (unsigned int i = 0; i < _events.size(); i++) {
				if (FD_ISSET(_fds[i], &rfds)) {
					/*
					 * 这里只处理_fds列表中的第一个fd上的事件，处理完成后就return了，下面有注释，主要
					 * 考虑action中可能会修改_events[]列表，对于后面的fd，会放到下一次循环中处理，由于
					 * 没有处理的fd上的事件，在下一次循环进入select时就会得到处理，所以，理论上，不会有
					 * 太大的延时，但是理论上，如果第一个fd上不停的有事件触发，那么后面的fd上的事件就可能
					 * 得不到处理，导致饥饿。这里可能有一定的隐患。
					 */
		    	_events[i]->action();
		    	/* The action may have removed / added event sources changing
		       our array, so leave the loop and handle other events the next
		       time we are called */
		    	return false;
				}
	    }
	    return false;
	}

ProcessLoop::run()->EventSources::wait_events()->XEventHandler::on_event():

	//在select监控到相关socket上的事件后，进入这里。这是使用Xlib处理XEvent的标准代码框架
	void XEventHandler::on_event()
	{
    XLockDisplay(x_display);
		// 检测是否有XEvent挂起
    while (XPending(&_x_display)) {
        XPointer proc_pointer;
        XEvent event;
				/*
				 * 从Client的input事件队列中取出下一个事件，如果没有，则会flush自己的output
				 * 队列(其中存放Client将要发给Xorg的Request)，然后阻塞等待下一个Event来。
				 * 由于前面已经通过XPending检测了事件，所以，这里理论上不会阻塞。
				 * XNextEvent是Xlib中用于处理XEvent消息的最常用的接口。其会将next event从input
				 * 队列中取出。相比XPeekEvent，XPeekEvent不会删除input队列中的数据。
				 */
        XNextEvent(&_x_display, &event);
				// 判断取出的事件是否跟窗口相关，不相关，则抛弃。
        if (event.xany.window == None) {
            continue;
        }
				// 对事件进行过滤，标准接口
				if (XFilterEvent(&event, None)) {
				    continue;
				}
				// 通过event中的window属性找到相应的context
        if (XFindContext(&_x_display, event.xany.window, _win_proc_context, &proc_pointer)) {
            /* When XIM + ibus is in use XIM creates an invisible window for
               its own purposes, we sometimes get a _GTK_LOAD_ICONTHEMES
               ClientMessage event on this window -> skip logging. */
            if (event.type != ClientMessage) {
							#ifndef USE_VIDEO_OVERLAYER
							LOG_WARN(
                    "Event on window without a win proc, type: %d, window: %u",
                    event.type, (unsigned int)event.xany.window);
							#endif
            }
            continue;
        }
        XUnlockDisplay(x_display);
				// 调用win_proc，进行实际的event处理
        ((XPlatform::win_proc_t)proc_pointer)(event);
        XLockDisplay(x_display);
    }
    XUnlockDisplay(x_display);
}

ProcessLoop::run()->EventSources::wait_events()->XEventHandler::on_event()->RedWindow_p::win_proc():

	// 窗口事件的实际处理接口
	void RedWindow_p::win_proc(XEvent& event)
	{
	    XPointer window_pointer;
	    RedWindow* red_window;
	    int res;
	    static int count = 0;

	    XLockDisplay(x_display);
	    res = XFindContext(x_display, event.xany.window, user_data_context, &window_pointer);
	    XUnlockDisplay(x_display);
	    if (res) {
	        THROW("no user data");
	    }
	    red_window = (RedWindow*)window_pointer;
			//根据event的类型进行相应处理
	    switch (event.type) {
	    case MotionNotify: {
	        SpicePoint size = red_window->get_size();
	        if (event.xmotion.x >= 0 && event.xmotion.y >= 0 &&
	            event.xmotion.x < size.x && event.xmotion.y < size.y) {
	            SpicePoint origin = red_window->get_origin();
	            red_window->get_listener().on_pointer_motion(event.xmotion.x - origin.x,
	                                                         event.xmotion.y - origin.y,
	                                                         to_red_buttons_state(event.xmotion.state));
	        }
	        break;
	    }
			// 键盘“按下”事件
	    case KeyPress:
					// 调用窗口定义的handle，并更新事件时间，设置窗口的wm_user_time属性，窗口管理器会使用该属性来设置窗口stack。
	        red_window->handle_key_press_event(*red_window, &event.xkey);
	        red_window->_last_event_time = event.xkey.time;
	        XChangeProperty(x_display, red_window->_win, wm_user_time,
	                        XA_CARDINAL, 32, PropModeReplace,
	                        (unsigned char *)&event.xkey.time, 1);
	        break;
			// 键盘“弹起”事件
	    case KeyRelease: {
	        RedKey key = to_red_key_code(event.xkey.keycode);
	        XEvent next_event;
	        XLockDisplay(x_display);
					/*
					 * 获取与本窗口相关的next event，用于检测是否在release事件之后是否连续出现了press
					 * 事件，如果是的化，就不处理这次的Release事件的了。
					 * 注意：这里的XCheckWindowEvent会从input事件队列中取出事件。所以，才用后面的
					 * XPutBackEvent将消息重新放回队列中
					 */
	        Bool check = XCheckWindowEvent(x_display, red_window->_win, ~long(0), &next_event);
	        XUnlockDisplay(x_display);
	        if (check) {
							/*
							 * 将消息重新放回队列中，由于前面的XCheckWindowEvent事件会取出事件，如果不
							 * 放回去，会丢事件，但这里没加锁，如果出现并发，理论上还是有一定的风险
							 */
	            XPutBackEvent(x_display, &next_event);
							// 如果下一个事件是KeyPress，则跳过本次处理
	            if ((next_event.type == KeyPress) &&
	                (event.xkey.keycode == next_event.xkey.keycode) &&
	                (event.xkey.time == next_event.xkey.time)) {
	                break;
	            }
	        }
	        if (key != REDKEY_KOREAN_HANGUL && key != REDKEY_KOREAN_HANGUL_HANJA) {
	            red_window->get_listener().on_key_release(key);
	        }
	        break;
	    }
	    ...
		}

后面的流程与本次故障无关，就不分析了。

为监控XEvent事件的实际到达情况，打点的位置即在wait_events函数中select函数之后的某个流程中，通过打点确认，在故障复现时，select只按时收到了KeyPress事件，KeyRelease事件没有按时收到，出现了延时。直接现象就是select没有及时被唤醒。

于是，问题焦点为：为何select没有及时监控到事件？

理论上有如下可能性：

1. 发送端(Xorg)没有及时发送相应事件。
2. select通道出了问题，即Xorg发送了，但由于通道问题，导致接收端没有被及时唤醒。这个通道涉及C库和内核。
3. Spice中通过select监控了多个fd，多个fd之间处理有一定的规则(见代码注释)，是否可能在特殊情况下出现问题。

后续的打点，依次验证了上述可能性。

## 阶段2：Xorg打点

这个阶段在Xorg中打点，确认Xorg是否及时发送了相应的事件。

理论上，Xorg与Client之间的事件交互也是通过socket，而且其自身也可能有一定的缓存机制。

先看看Xorg中的相关代码流程，Xorg从事件分发到最终将事件发送给Client的整体代码流程如下：

	Dispatch  //Xorg主循环
		ProcessInputEvents
			mieqProcessDeviceEvent
				ProcessKeyboardEvent
					AccessXFilterPressEvent
						XkbProcessKeyboardEvent //这里开始是Xcb的相关流程
							XkbHandleActions
								dev->public.processInputProc((InternalEvent *) event, tmpdev);
									ProcessOtherEvent // 这里开始将事件发送给Client
										DeliverRawEvent
											DeliverEventToInputClients  
												TryClientEvents // 这里开始是本故障相关的主要流程
													WriteEventsToClient
														WriteToClient
															FlushClient
																_XSERVTransWritev //这里进入XTrans相关流程
																	writev(socket,...) // 最终写入相应的套接字

流程很复杂，这里未提及从内核到Xorg的事件传递流程和机制，之前分析过，可以参考相应的文章。

其中，关键的留存为从TryClientEvents函数开始的部分，过程中也有相应的缓存机制，可能会出现延迟的情况，分别看看关键的代码：

TryClientEvents():

	// 将xEvent发送到指定的Client
	int
	TryClientEvents(ClientPtr client, DeviceIntPtr dev, xEvent *pEvents,
	                int count, Mask mask, Mask filter, GrabPtr grab)
	{
	    int type;
			...
	    type = pEvents->u.u.type;
			// 鼠标移动事件
	    if (type == MotionNotify) {
	        if (mask & PointerMotionHintMask) {
	            if (WID(dev->valuator->motionHintWindow) ==
	                pEvents->u.keyButtonPointer.event) {
	#ifdef DEBUG_EVENTS
	                ErrorF("[dix] \n");
	                ErrorF("[dix] motionHintWindow == keyButtonPointer.event\n");
	#endif
	                return 1;       /* don't send, but pretend we did */
	            }
	            pEvents->u.u.detail = NotifyHint;
	        }
	        else {
	            pEvents->u.u.detail = NotifyNormal;
	        }
	    }
	    else if (type == DeviceMotionNotify) {
	        if (MaybeSendDeviceMotionNotifyHint((deviceKeyButtonPointer *) pEvents,
	                                            mask) != 0)
	            return 1;
	    }
			// 键盘“按下”事件
	    else if (type == KeyPress) {
					// 判断是否是重复按键，即用户按下不释放的情况，这种情况下，会自动repeat相应的事件
	        if (EventIsKeyRepeat(pEvents)) {
	            if (!_XkbWantsDetectableAutoRepeat(client)) {
	                xEvent release = *pEvents;

	                release.u.u.type = KeyRelease;
	                WriteEventsToClient(client, 1, &release);
	#ifdef DEBUG_EVENTS
	                ErrorF(" (plus fake core release for repeat)");
	#endif
	            }
	            else {
	#ifdef DEBUG_EVENTS
	                ErrorF(" (detectable autorepeat for core)");
	#endif
	            }
	        }

	    }
			...
			// 这里继续向下，实现event的发送
	    WriteEventsToClient(client, count, pEvents);
	#ifdef DEBUG_EVENTS
	    ErrorF("[dix]  delivered\n");
	#endif
	    return 1;
	}

TryClientEvents->WriteEventsToClient->WriteToClient():

	// 这个函数中可能阻塞
	int
	WriteToClient(ClientPtr who, int count, const void *__buf)
	{
			OsCommPtr oc;
			// 用户控制与客户端连接后的数据传送
	    ConnectionOutputPtr oco;
	    int padBytes;
	    const char *buf = __buf;
			...
			// 没有oco的话重新创建并初始化
	    if (!oco) {
	        if ((oco = FreeOutputs)) {
	            FreeOutputs = oco->next;
	        }
	        else if (!(oco = AllocateOutputBuffer())) {
	            if (oc->trans_conn) {
	                _XSERVTransDisconnect(oc->trans_conn);
	                _XSERVTransClose(oc->trans_conn);
	                oc->trans_conn = NULL;
	            }
	            MarkClientException(who);
	            return -1;
	        }
	        oc->output = oco;
	    }
			// 对齐处理
	    padBytes = padding_for_int32(count);
			...
			// 这里为关键的流程。
	    if (oco->count == 0 || oco->count + count + padBytes > oco->size) {
					/*
					 * 如果进入这个分支，表示立即发送。进入该分支的条件为(满足之一即可)：
					 * 1. oco->count == 0，这个可以在FlushClient的流程中看到，当flush(即write)
					 *    数据不完整，即socket出现阻塞，不能一次event对应的数据发送完成时，会设置该变量，
					 *		表明上次发送未完成。如果是这样，这里就不会立即发送。
					 * 2. 当前oco中缓存的数据溢出了。即数据大于其缓冲区的大小了。此时，必须flush了，否则
					 *    后面的数据塞不进来了。
					FD_CLR(oc->fd, &OutputPending);
	        if (!XFD_ANYSET(&OutputPending)) {
	            CriticalOutputPending = FALSE;
	            NewOutputPending = FALSE;
	        }

	        if (FlushCallback)
	            CallCallbacks(&FlushCallback, NULL);

	        return FlushClient(who, oc, buf, count);
	    }
			/*
			 * 走到这里说明event不会被立即发送，而是先缓存起来，直到下一次走到上面的if分支，满足其两个
			 * 条件之一，才能被发送出去。
			 * 缓存后一起发送的目的自然是考虑效率的因素
			 */

	    NewOutputPending = TRUE;
	    FD_SET(oc->fd, &OutputPending);
	    memmove((char *) oco->buf + oco->count, buf, count);
	    oco->count += count;
	    if (padBytes) {
	        memset(oco->buf + oco->count, '\0', padBytes);
	        oco->count += padBytes;
	    }
	    return count;
	}

TryClientEvents->WriteEventsToClient->WriteToClient->FlushClient():

	//这个函数也可能阻塞
	int
	FlushClient(ClientPtr who, OsCommPtr oc, const void *__extraBuf, int extraCount)
	{
	    ConnectionOutputPtr oco = oc->output;
	    int connection = oc->fd;
	    XtransConnInfo trans_conn = oc->trans_conn;
	    struct iovec iov[3];
	    static char padBuffer[3];
	    const char *extraBuf = __extraBuf;
	    long written;
	    long padsize;
	    long notWritten;
	    long todo;

		...
					//为writev准备数据
					InsertIOV((char *) oco->buf, oco->count)
	        InsertIOV((char *) extraBuf, extraCount)
	        InsertIOV(padBuffer, padsize)

	        errno = 0;
					// 调用XTrans接口，向相应的Socket中写入数据
	        if (trans_conn && (len = _XSERVTransWritev(trans_conn, iov, i)) >= 0) {
	            written += len;
	            notWritten -= len;
	            todo = notWritten;
	        }
	        else if (ETEST(errno)
	#ifdef SUNSYSV                  /* check for another brain-damaged OS bug */
	                 || (errno == 0)
	#endif
	#ifdef EMSGSIZE                 /* check for another brain-damaged OS bug */
	                 || ((errno == EMSGSIZE) && (todo == 1))
	#endif
	            ) {
								// socket出现阻塞，此时需要暂缓发送事件
	            /* If we've arrived here, then the client is stuffed to the gills
	               and not ready to accept more.  Make a note of it and buffer
	               the rest. */
	            FD_SET(connection, &ClientsWriteBlocked);
	            AnyClientsWriteBlocked = TRUE;

	            if (written < oco->count) {
	                if (written > 0) {
	                    oco->count -= written;
	                    memmove((char *) oco->buf,
	                            (char *) oco->buf + written, oco->count);
	                    written = 0;
	                }
	            }
	            else {
	                written -= oco->count;
	                oco->count = 0;
	            }

	            if (notWritten > oco->size) {
	                unsigned char *obuf;

	                obuf = (unsigned char *) realloc(oco->buf,
	                                                 notWritten + BUFSIZE);
	                if (!obuf) {
	                    _XSERVTransDisconnect(oc->trans_conn);
	                    _XSERVTransClose(oc->trans_conn);
	                    oc->trans_conn = NULL;
	                    MarkClientException(who);
	                    oco->count = 0;
	                    return -1;
	                }
	                oco->size = notWritten + BUFSIZE;
	                oco->buf = obuf;
	            }

	            /* If the amount written extended into the padBuffer, then the
	               difference "extraCount - written" may be less than 0 */
	            if ((len = extraCount - written) > 0)
	                memmove((char *) oco->buf + oco->count,
	                        extraBuf + written, len);
							// 这里设置了count，会导致上一级函数中出现阻塞，不立即flush
	            oco->count = notWritten;    /* this will include the pad */
	            /* return only the amount explicitly requested */
	            return extraCount;
	        }
					...
	}

后面XTrans相关的代码比比较复杂，这里就不继续列举分析了，有兴趣可以自己研究，代码在/usr/include/X11/XTrans目录，没有在Xorg的源码包中。

在TryClientEvents->WriteEventsToClient->WriteToClient->FlushClient->XTrans... 相关关键留存中打点，确认故在复现是相应的KeyRelease事件已经成功并及时发送，没有出现阻塞的情况。打点日志片段如下：

	[   457.343] [dix] Event([2, 38], mask=0x2a807f), client=4++++++XTRANS_SEND_FDS:
	 ciptr=81b8dd70, send_fds=0, buf=bfa04ab8, size=1 //“按下”事件
	[   457.343] ++++++SocketWritev: ciptr=81b8dd70, fd=22, buf=bfa04ab8, size=1
	[   457.343] [++++++] pClient->swapped=0 write length=32, pclient=81be6c98, ret=32.
	[   457.343] [dix]  delivered //发送成功
	[   457.343] [dix] Event([2, 38], mask=0x620000), client=7 filtered //另一个“按下”事件被过滤掉
	[   457.344] [dix] Event([28, 0], mask=0x2a807f), client=4 filtered
	[   457.344] [dix] Event([28, 0], mask=0x620000), client=7++++++XTRANS_SEND_FDS:
	 ciptr=81311e78, send_fds=0, buf=bfa05f98, size=1
	[   457.344] ++++++SocketWritev: ciptr=81311e78, fd=25, buf=bfa05f98, size=1
	[   457.344] [++++++] pClient->swapped=0 write length=32, pclient=8160b2b8, ret=32.
	[   457.344] [dix]  delivered //中间会产生两个28类型的按键，不用关心
	[   457.431] [dix] Event([3, 38], mask=0x2a807f), client=4[++++++]Flush: oco->count=0, oco->size=4096, padBytes=0,buf=817ca3c8
	[   457.431] ++++++XTRANS_SEND_FDS:
	 ciptr=81b8dd70, send_fds=0, buf=bfa04a68, size=1
	[   457.431] ++++++SocketWritev: ciptr=81b8dd70, fd=22, buf=bfa04a68, size=1
	[   457.431] [++++++] pClient->swapped=0 write length=32, pclient=81be6c98, ret=32.
	[   457.431] [dix]  delivered // “弹起”事件，发送成功
	[   457.431] [dix] Event([3, 38], mask=0x620000), client=7 filtered // 另一个“弹起”事件被过滤

从打点日志看，每次按键会产生6个事件：2个“KeyPress”(其中一个被过滤)，2个“28”，2个“KeyRelease”（其中一个被过滤），28不用关注。  
从日志看，KeyRelease按键的却发送成功了，即被正确全部写入了socket，大小是32个字节，也没有发生阻塞。  
从时间看，KeyRelease的时间和KeyPress的时间相差100ms左右，没有任何延时。  
至此，可以肯定Xorg已经及时奖KeyRelease时间发送出去。

## 阶段3：Kernel打点

既然Xorg发送事件成功，那么需要进一步确认，事件“通道”(select)是否有问题，这个通道主要涉及两个方面：  
1. C库
2. 内核  
要确认问题，最直接的方法就是在内核中打点了，先跳过C库。  
打点前，还是需要分析情况相关代码流程，select相关的流程涉及两个方面：
1. 发送端
2. 接收端  

### 发送端的流程

主要是从writev开始，一直到与select相关的部分，整体上看，就是对socket的write过程。  
了解内核的TX应该都明白，这是个比较复杂的过程，首先需要确认socket的类型，不同的socket，其write流程差别很大。  
对于Xorg和Client之间通信用的socket，是通过XTrans相关接口实现的，整体框架比较复杂，创建该socket的入口用户态接口使大家熟悉的XOpenDisplay，大致实现流程为：

	XOpenDisplay
		_X11TransOpenCOTSClient
			TRANS(Open)
				TRANS(SocketOpenCOTSClient)
		_X11TransConnectDisplay

TRANS(SocketOpenCOTSClient)最终会根据不同的环境，创建不同的socket，比如如果是本机，则创建STREAM类型的Unix domain socket；如果Xorg和Client不在同一台主机上，这创建TCP类型的socket。

对于本案例，对应的socket为STREAM类型的Unix domain socket，其对应write接口为：  
unix_stream_sendmsg（至于如何从用户态的write走到这里，可以自己了解下～）  
其对应的主要代码流程如下：

	static int unix_stream_sendmsg(struct socket *sock, struct msghdr *msg,
				       size_t len)
	{
		...
		err = scm_send(sock, msg, &scm, false);
		if (err < 0)
			return err;

		err = -EOPNOTSUPP;
		if (msg->msg_flags&MSG_OOB)
			goto out_err;

		if (msg->msg_namelen) {
			err = sk->sk_state == TCP_ESTABLISHED ? -EISCONN : -EOPNOTSUPP;
			goto out_err;
		} else {
			err = -ENOTCONN;
			// unix domain socket有两端，即一对，需要找到另一端的socket
			other = unix_peer(sk);
			if (!other)
				goto out_err;
		}

		if (sk->sk_shutdown & SEND_SHUTDOWN)
			goto pipe_err;
		// 这里开始发送，当sent(已发送的字节)>=len
		while (sent < len) {
			size = len - sent;

			/* Keep two messages in the pipe so it schedules better */
			size = min_t(int, size, (sk->sk_sndbuf >> 1) - 64);

			/* allow fallback to order-0 allocations */
			size = min_t(int, size, SKB_MAX_HEAD(0) + UNIX_SKB_FRAGS_SZ);

			data_len = max_t(int, 0, size - SKB_MAX_HEAD(0));

			data_len = min_t(size_t, size, PAGE_ALIGN(data_len));
			//分配skb
			skb = sock_alloc_send_pskb(sk, size - data_len, data_len,
						   msg->msg_flags & MSG_DONTWAIT, &err,
						   get_order(UNIX_SKB_FRAGS_SZ));
			if (!skb)
				goto out_err;

			/* Only send the fds in the first buffer */
			err = unix_scm_to_skb(&scm, skb, !fds_sent);
			if (err < 0) {
				kfree_skb(skb);
				goto out_err;
			}
			fds_sent = true;
			//设置skb
			skb_put(skb, size - data_len);
			skb->data_len = data_len;
			skb->len = size;
			// 拷贝数据
			err = skb_copy_datagram_from_iter(skb, 0, &msg->msg_iter, size);
			if (err) {
				kfree_skb(skb);
				goto out_err;
			}
			...
			/*
			 * 将skb直接放到other(即对端的socket)的接收队列中，非常直接。如果走协议栈的话，这个
			 * 过程会非常复杂。可见，通过unix domain socket来进行进程间通信(IPC)的效率是非常高的，
			 * 没有复杂多余的过程
			*/
			skb_queue_tail(&other->sk_receive_queue, skb);
			unix_state_unlock(other);
			/*
			 * 调用相应钩子(默认为sock_def_readable),主要目的就是唤醒对端等待队列上的进程(等待数据到来的进程)
			 * 典型的就是selecet和poll，本例关注的select的进程的唤醒过程就在这里实现。
			 */
			other->sk_data_ready(other);
			sent += size;
		}

unix_stream_sendmsg->sock_def_readable():

	static void sock_def_readable(struct sock *sk)
	{
		struct socket_wq *wq;

		rcu_read_lock();
		// 获取socket上的等待队列，每个socket都有一个，等待该socket数据的进程都会挂到这个等待队列中
		wq = rcu_dereference(sk->sk_wq);
		// 判断该等待队列中是否已有sleep等待的进程
		if (skwq_has_sleeper(wq))
			// 如果有的话，就直接唤醒相关进程
			wake_up_interruptible_sync_poll(&wq->wait, POLLIN | POLLPRI |
							POLLRDNORM | POLLRDBAND);
		/*
		 * 如果没有的话，本来就不需要做唤醒了，但是有一种特殊情况，就算async，即通过SIGIO的异步
		 * 通知机制，这种情况比较少。比如键盘事件从内核edev驱动发送给Xorg就是通过这个机制，但与
		 * 这里无关
		 */
		sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
		rcu_read_unlock();
	}

wake_up_interruptible_sync_poll及sk_wake_async的具体流程大家可以看看代码

### 接收端流程

接收端流程即select的流程，在内核中，基本是从do_select开始的，其中逻辑比较简单，其中最主要的就是调用了底层驱动实现的poll钩子，对于unix domain socket套接字来说，其实现的对应的钩子为：unix_poll

do_select主要代码为(我们关心的部分)：

	static int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)
	{
		...
		// 初始化等待列表
		poll_initwait(&table);
		wait = &table.pt;
		// 如果设置的超时时间为0,则select就不会等待(睡眠)，如果没有查询到，就直接返回
		if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
			wait->_qproc = NULL;
			timed_out = 1;
		}
		...
		// 循环查询fd_set(所有被监控fd)中所有的fd，检查read、write、exception的状态
		for (;;) {
			unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
			bool can_busy_loop = false;

			inp = fds->in; outp = fds->out; exp = fds->ex;
			rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

			for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
				unsigned long in, out, ex, all_bits, bit = 1, mask, j;
				unsigned long res_in = 0, res_out = 0, res_ex = 0;

				in = *inp++; out = *outp++; ex = *exp++;
				all_bits = in | out | ex;
				if (all_bits == 0) {
					i += BITS_PER_LONG;
					continue;
				}

				for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {
					struct fd f;
					if (i >= n)
						break;
					if (!(bit & all_bits))
						continue;
					f = fdget(i);
					if (f.file) {
						const struct file_operations *f_op;
						f_op = f.file->f_op;
						mask = DEFAULT_POLLMASK;
						if (f_op->poll) {
							wait_key_set(wait, in, out,
								     bit, busy_flag);
							/*
							 * 这里调用底层驱动实现的poll接口，不同的文件实现不同，依赖倒置。所有需要支持
							 * select的文件，都需要实现该接口
							 */
							mask = (*f_op->poll)(f.file, wait);
						}
						fdput(f);
						/*
						 * POLLIN_SET\POLLOUT_SET\POLLEX_SET分别对应3个集合，当底层poll接口中查询
						 * 到相应事件后，会设置相应的标记位，比如如果fd上有数据可以read，则会设置POLLIN标记位
						 * 通过如下3个判断后，一旦有事件，并与原先指定的监控集与以后，就会执行retval++
						 */
						if ((mask & POLLIN_SET) && (in & bit)) {
							res_in |= bit;
							retval++;
							wait->_qproc = NULL;
						}
						if ((mask & POLLOUT_SET) && (out & bit)) {
							res_out |= bit;
							retval++;
							wait->_qproc = NULL;
						}
						if ((mask & POLLEX_SET) && (ex & bit)) {
							res_ex |= bit;
							retval++;
							wait->_qproc = NULL;
						}
						// 监控到了事件，就要退出do_select的事件监测循环了，就不会sleep
						/* got something, stop busy polling */
						if (retval) {
							can_busy_loop = false;
							busy_flag = 0;

						/*
						 * only remember a returned
						 * POLL_BUSY_LOOP if we asked for it
						 */
						} else if (busy_flag & mask)
							can_busy_loop = true;

					}
				}
				if (res_in)
					*rinp = res_in;
				if (res_out)
					*routp = res_out;
				if (res_ex)
					*rexp = res_ex;
				cond_resched();
			}
			wait->_qproc = NULL;
			// 这里退出循环
			if (retval || timed_out || signal_pending(current))
				break;
			if (table.error) {
				retval = table.error;
				break;
			}
			// 监测超时，超时后，也不睡眠了
			/* only if found POLL_BUSY_LOOP sockets && not out of time */
			if (can_busy_loop && !need_resched()) {
				if (!busy_start) {
					busy_start = busy_loop_current_time();
					continue;
				}
				if (!busy_loop_timeout(busy_start))
					continue;
			}
			busy_flag = 0;

			/*
			 * If this is the first loop and we have a timeout
			 * given, then we convert to ktime_t and set the to
			 * pointer to the expiry value.
			 */
			if (end_time && !to) {
				expire = timespec64_to_ktime(*end_time);
				to = &expire;
			}
			// 主动调度，开始sleep了，有超时
			if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
						   to, slack))
				timed_out = 1;
		}

		poll_freewait(&table);

		return retval;
	}


unix_poll主要代码：

	static unsigned int unix_poll(struct file *file, struct socket *sock, poll_table *wait)
	{
		struct sock *sk = sock->sk;
		unsigned int mask;
		// 将当前进程加入socket的等待队列
		sock_poll_wait(file, sk_sleep(sk), wait);
		mask = 0;
		// 出现了异常，也会导致select退出
		/* exceptional events? */
		if (sk->sk_err)
			mask |= POLLERR;
		if (sk->sk_shutdown == SHUTDOWN_MASK)
			mask |= POLLHUP;
		if (sk->sk_shutdown & RCV_SHUTDOWN)
			mask |= POLLRDHUP | POLLIN | POLLRDNORM;
		/*
		 * 判断fd(socket)是否可读，注意：其判断标准为socket的接收队列是否为空，意思是只要接收队列
		 * 中有数据，就是可读的。结合前面的sendmsg流程，可以确认在sendmsg之后，肯定是会将skb放入
		 * 接收队列的。所以，理论上，只要sendmsg，即write成功，那么select就一定能监控到相应的事件
		 */
		/* readable? */
		if (!skb_queue_empty(&sk->sk_receive_queue))
			mask |= POLLIN | POLLRDNORM;

		/* Connection-based need to check for termination and startup */
		if ((sk->sk_type == SOCK_STREAM || sk->sk_type == SOCK_SEQPACKET) &&
		    sk->sk_state == TCP_CLOSE)
			mask |= POLLHUP;

		/*
		 * we set writable also when the other side has shut down the
		 * connection. This prevents stuck sockets.
		 */
		if (unix_writable(sk))
			mask |= POLLOUT | POLLWRNORM | POLLWRBAND;

		return mask;
	}

注意：其判断标准为socket的接收队列是否为空，意思是只要接收队列中有数据，就是可读的。结合前面的sendmsg流程，可以确认在sendmsg之后，肯定是会将skb放入接收队列的。所以，理论上，只要sendmsg，即write成功，那么select就一定能监控到相应的事件。

所以，理论上，基本也可以排除内核的问题。

由于其他地方也没有发现疑点，保险起见，还是需要打点确认。在相关的关键流程中都打了点，可以确认数据确实发送成功了，但是select确实没有监控到相应的事件（事件非常多，需要根据大小、socket来过滤，比较费劲）

需要进一步打点，确认socket接收队列中的skb的状态，上述的打点已经确认skb已经放入接收队列了，但当select时，却发现接收队列为空了。只有一种可能：在这个过程中有人把skb取走了。要确认该疑点，需要在取走skb的流程中打点，对应的函数(宏)为：skb_unlink

	void skb_unlink(struct sk_buff *skb, struct sk_buff_head *list)
	{
		unsigned long flags;

		spin_lock_irqsave(&list->lock, flags);
		__skb_unlink(skb, list);
		spin_unlock_irqrestore(&list->lock, flags);
	}

在这个函数中加入打印，并通过dump_stack()打印堆栈，即可确认是什么流程取走了相关skb。打点结果如：

	Dec 12 16:57:11 localhost kernel: [  106.560092] CPU: 0 PID: 5497 Comm: spicec_V4.01.01 Tainted: G           OE   4.10.11 #16
	Dec 12 16:57:11 localhost kernel: [  106.560093] Hardware name:xxx, BIOS Z09.005 04/06/2017
	Dec 12 16:57:11 localhost kernel: [  106.560095] Call Trace:
	Dec 12 16:57:11 localhost kernel: [  106.560099]  show_stack+0x28/0x60
	Dec 12 16:57:11 localhost kernel: [  106.560102]  dump_stack+0x5b/0x80
	Dec 12 16:57:11 localhost kernel: [  106.560105]  skb_unlink+0x6f/0x90
	Dec 12 16:57:11 localhost kernel: [  106.560108]  unix_stream_read_generic+0x585/0x6d0
	Dec 12 16:57:11 localhost kernel: [  106.560117]  unix_stream_recvmsg+0x47/0x60
	Dec 12 16:57:11 localhost kernel: [  106.560123]  ? unix_state_double_lock+0x60/0x60
	Dec 12 16:57:11 localhost kernel: [  106.560128]  sock_recvmsg+0x3f/0x50
	Dec 12 16:57:11 localhost kernel: [  106.560134]  ___sys_recvmsg+0x9c/0x120
	Dec 12 16:57:11 localhost kernel: [  106.560140]  ? SyS_setsockopt+0xb0/0xb0
	Dec 12 16:57:11 localhost kernel: [  106.560148]  ? sock_sendmsg+0x45/0x50
	Dec 12 16:57:11 localhost kernel: [  106.560154]  ? update_curr+0x1be/0x2a0
	Dec 12 16:57:11 localhost kernel: [  106.560159]  ? __fget_light+0x28/0x60
	Dec 12 16:57:11 localhost kernel: [  106.560164]  __sys_recvmsg+0x3e/0x70
	Dec 12 16:57:11 localhost kernel: [  106.560169]  SyS_recvmsg+0x16/0x20
	Dec 12 16:57:11 localhost kernel: [  106.560175]  SYSC_socketcall+0x108/0x2e0

可以看出是spice自己调用了recvmsg系统调用取走了相应的事件，有可能是并发的spice上下文，也有可能是在进入select之前的串行操作。

## 阶段4： spice再次打点

至此，问题焦点再次回到了spice自身。由于内核中打印只能确认从系统调用入口到内核中的流程，不能确认用户态的留存，还需要确认Spice在用户态的流程。

由于内核中不方便打印用户态的堆栈，而且故障发生看似有比较严格的时序要求，难以捕获到故障发生的现场。对于这种问题，有两种思路：

1. 在spice中打点，在所有可能调用recvmsg的地方打上点，如果走到了就能抓到。
2. 分析代码。这是分析所有问题的终极解决办法。

于是，再次在Spice中打点，由于其针对的是与Xorg的连接的Socket，所以不太可能通过其他接口，基本只能通过Xlib的接口读取，分析了Xlib相关的接口代码和手册，可能读取Socket中的xEvent事件的接口很多，包括XPeekEvent、XNextEvent、XCheckxxx之类的常规接口，这类接口通常会直接读取相应事件，并进行处理。

于是，在Spice中调用这类接口的地方都打上了点，复现抓取后，发现都没有走到。接下来，只能继续分析代码了。

## 阶段5： 理论和代码分析

再次分析了Xorg和Client的通信机制和相关协议，发现Xorg和Client之间的通信时，在Client侧，实际每个Client自己维护了两个队列：Input和Output队列。

Output队列用于存放Client发送给Xorg的Request。这里暂不关注。

Input队列用于存放从Xorg发送过来的xEvent事件，因为Client不一定来得及及时处理，所以需要有这样的缓存机制，也能提高效率。也就是说，有两个地方能存放xEvent事件：

1. Unix domain Socket的接收队列。这个能通过select监控。
2. Client的Input队列，这个select就监控不到了。但时XPending和XNextEvent之类的接口能监控到。

理论上，如果进入select时，相应的xEvent已经从socket的接收队列移入到Client的接收队列，那么select就监控不到了，而XNextEvent就能取到，如此就能解释当前的问题现象。

需要继续分析，哪些操作会从Socket中取xEvent事件，放入Input队列，分析了Xlib相关代码，这个操作分两个动作：

1. 从socket中取数据，对应的接口为：poll_for_event，最终通过xcb的底层接口去recv。
2. 放入Input队列，对应的底层接口为：_XEnq。

进行这两个动作的Xlib接口也很多，包括前面提到“取事件”的接口，还包括一些看似“不需要取事件”的接口，比如XFlush和XSync，这类接口会经常使用。XSync进行相关操作的主要代码流程如下：

	XSync
		_XReply
			poll_for_event
			handle_response
				_XEnq

具体代码，这里就不展示了。

# 故障出现流程

再结合Spice的代码和打点日志分析，基本可以确认问题出现的流程了：

1. spice主循环中通过select轮询多个fd，其中一个时与Xorg连接的socket，另一个是trigger，是一个管道，用于唤醒spice。
2. 用户按下键盘按键，首先触发KeyPress事件。该事件正常处理。
3. 约100ms之后，触发KeyRelease事件，Xorg正常将事件写入socket
4. 在KeyRelease事件到来之前，select监控的trigger fd上监控到了相应的事件：SPICE_SCREEN_UPDATE_EVENT，该事件会触发spice更新当前的界面显示，会进行一些绘图操作(底层调用的pixman库)，其中(可能)会调用XFlush或XSync之类的操作。
5. 在调用XFlush之类的操作之前，刚好KeyRelease事件被写入了Socket接收队列中，此时XFlush之类的操作会从Socket接收队列中取出xEvent事件，并将其放入Input队列。
6. spice重新进入主循环，进入select，此时由于socket中的事件已经被取走了，所以，就查询不到事件，因此就一直睡眠了。

# 结论

利用select来监听XEvent的方法并不可靠。问题在于Spice中监控XEvent事件的方法问题。

# 解决方法

在使用Xlib编程时，监控Xorg事件(XEvent)的标准方法应该还是结合XPending+XNextEvent的方法，示例代码如spice中on_event()中的代码，大致框架为：
 while (XPending(dsp)) {
        ...
        XNextEvent();
        XFilterEvent()
        handle_event();
    }
如此，不会漏掉任何事件。
但这种方法不能实现多路复用(不能同时监控多个fd)，也不能解决超时的问题(如果一直没有事件来，会一致阻塞)。

所以，针对Spice需要使用多路复用和解决超时问题的场景下，建议使用XPending+select的方法，即在select之前先通过XPending来判断是否有事件。
