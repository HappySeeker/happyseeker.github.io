---
layout: post
title:  "窗口管理器Marco--窗口stack错乱问题分析"
date:   2016-11-23 11:35:06
author: HappySeeker
categories: Graphic
---

一直想写些关于窗口管理器相关的内容，一直每时间深入分析相关代码。最近刚好遇到一个诡异的QT界面消失问题，顺便分析了相关代码。本文主要针对该问题相关内容进行梳理，其它部分后面有时间再整理吧。

# 问题

有个QT应用，界面是一个悬浮菜单，用于实现窗口切换。在使用过程中，偶尔会出现菜单无故消失的现象，而相关进程状态正常，无任何异常上报，只是界面消失了。

# 分析
## 从现象开始

进程状态正常，说明应用程序本身并无异常，问题应该出在界面显示部分，从现象来看，可以想象如下可能性：

1. 应用程序自身代码异常导致界面消失，比如在可能的流程中调用了XUnmapWindow之类的接口。分析了相关代码，并查看故障时进程的堆栈，确认可以排除这种可能性。
2. 在X11环境中，谁负责显示？当然是Xorg，不能排除Xorg自身的显示出问题了，这点需要进一步分析确认。
3. 可能是其它窗口覆盖了问题应用的窗口。谁负责窗口的层叠次序(stack)? 当然是窗口管理器。那么，问题可能出在窗口管理器上。另一方面，通过在环境中替换默认的窗口管理器(Marco)为Compiz后，问题不再出现，说明Marco自身问题的可能性比较大。

## 从窗口显示的本质出发

故障现象是：原本显示的窗口突然不见了，而Xorg负责窗口的显示，那么就从Xorg显示窗口的流程进行分析。

对应X11图形应用程序来说，如果使用Xlib的接口直接编程(其它工具GTK/QT之类的直接或间接基于Xlib/XCB接口实现)，要创建并显示一个窗口，通常就使用基本的两个接口：

    XCreateWindow() //创建窗口，输出为窗口ID
    XMapWindow() //显示窗口

即先创建窗口，然后使用map接口显示。

在创建窗口后，会得到标识窗口的ID，该ID在Xorg中具有唯一性。

在本问题中，初步的定位思路：由于进程状态正常，消失的窗口实际存在，其ID也应该可以通过Xlib相关接口查询到，可以现获取其ID，然后直接通过调用XMapWindow()接口来显示，确认是否可以显示出来，如果可以，则可用基本排除是Xorg自身的问题(即可能性2)，那么问题范围基本可以缩小到Marco上。

### 如何显示存在的窗口？

接下来需要解决两个问题：

1. 如何获取存在窗口的ID？ Xlib中提供了接口XQueryTree()，可以查询以指定窗口为根窗口所有children窗口，意味着我们可以通过该接口获取系统中所有的窗口(以root窗口为根，进行查询)。但如何从这么多窗口中挑出自己需要的窗口呢？每个窗口都有自己的属性，比如Name，根据Name进行匹配就可以了，方法看起来有点笨，但实用即可。
2. 如何显示指定窗口？简单，如前面所说，XMapWindow()即可，但在此之前，还需要将需要显示窗口raise到最前端，需要使用XMapWindow()。如此即可。

解决上述两个问题后，可以直接根据窗口的名称来显示指定窗口。示例代码如下：
    #include <X11/Xlib.h>
    #include <X11/Xutil.h>
    #include <X11/Xatom.h>
    #include <unistd.h>
    #include <string.h>
    #include <stdlib.h>
    #include <stdio.h>
    //#include "xwinctl.h"

    static Window currWindow;
    static Window rootWindow;
    static Display *dsp;
    static Atom wm_state;
    static Atom wm_state_above;
    static Atom wm_state_fullscreen;
    static char* getWindowName(Window win)
    {
      int status;
      char **list;
      int i = 0;
      XTextProperty wmName;
      char *name = NULL;

      if (NULL  == dsp){
          return NULL;
      }

      status = XGetWMName(dsp, win, &wmName);
      if ((status) && (wmName.value) && (wmName.nitems)){
          status = XmbTextPropertyToTextList(dsp, &wmName, &list, &i);
          if ((status >= Success) && i && (*list)){
              name = (char*)strdup(*list);
          }
      }

      return name;   
    }

    static void findWindow(Window root, char *winName)
    {
      char *tempWinName = NULL;
      int status;
      Window parentWindow;
      Window *childWindow;
      unsigned int numChildren;
      unsigned int i;

      tempWinName = getWindowName(root);
      if(tempWinName){
          if (strstr(tempWinName, winName)){
              currWindow = root;
          }
          free(tempWinName);
      }

      status = XQueryTree(dsp, root, &root, &parentWindow, &childWindow, &numChildren);
      if (0 == status)
          return;

      if (0 == numChildren)
          return;

      for (i = 0; i < numChildren; i++){
          findWindow(childWindow[i], winName);
      }
      XFree((char*)childWindow);
    }

    int createWindowHandle(char *winName)
    {
      int screen = -1;

      if (NULL == winName){
          return -1;
      }

      dsp = XOpenDisplay(NULL);
      if (NULL == dsp){
          return -1;
      }

      screen = DefaultScreen(dsp);
      printf("input winName is : %s\n", winName);
      printf("currWindow is : %d\n", currWindow);
      rootWindow = RootWindow(dsp, screen);
      findWindow(rootWindow, winName);
      wm_state = XInternAtom(dsp, "_NET_WM_STATE", False);
      wm_state_above = XInternAtom(dsp, "_NET_WM_STATE_ABOVE", False);
      wm_state_fullscreen = XInternAtom(dsp, "_NET_WM_STATE_FULLSCREEN", False);
      if (0 != currWindow){
          return currWindow;
      }

      return -1;
    }

    void showWindow()
    {
      Atom state[2];
      state[0] = wm_state_fullscreen;
      state[1] = wm_state_above;
      //XChangeProperty(dsp, currWindow, wm_state, XA_ATOM, 32, PropModeReplace,
      //                  (unsigned char*)state, 2);
      XRaiseWindow(dsp, currWindow);
      XMapWindow(dsp, currWindow);
      XFlush(dsp);
    }

    void hideWindow()
    {
      XUnmapWindow(dsp, currWindow);
      XFlush(dsp);
    }

    int main(int argc, char* argv[]) {
      char* winName = argv[1];
      int hide = atoi(argv[2]);
      createWindowHandle(winName);
      if(hide != 1)
        showWindow();
      else
        hideWindow();	 

    }

代码编译，需要手工指定X11依赖库，编译命令如：

    gcc -lX11 show-winow.c -o show-winow

编译后的程序使用方法：使用窗口名称作为第一个参数，第二个参数标识是否需要hide窗口，比如需要显示名为Test的窗口，执行的命令为：

    ./show-winow Test 0

注意：有些应用程序在创建窗口时没有为其指定名称，此时无法使用该程序来显示窗口，建议修改程序，为窗口加上名称。

### 显示结果

通过上诉命令显示消失的窗口，结果显示成功，说明窗口和Xorg本身并没有问题，那问题应该就在窗口管理器Marco上了。

## Marco相关分析
### 窗口Stack

窗口Stack，即窗口叠放的层次关系，在同一个图形界面环境中，有许多的应用程序可用一起打开，每个应用程序可能会有多个窗口，不可能让每个窗口都显示到独立的区域，所有的窗口都共享同一个桌面空间，那么必然会有窗口叠放顺序的问题，谁在前，谁在后。显然，放在前面(Above)的窗口必然会遮挡后面(Below)的窗口。这种顺序也称Z序。

Xorg自身并不管理窗口的Z序(但其持有最终的窗口Stack)，其只是简单的将新创建的窗口放置到最前面，每次如此，然后被遮挡窗口的内容就一直无法为用户所见。可以将系统中的窗口管理器进程(如marco)kill掉，并mv掉其可执行程序(比如:mv /usr/bin/marco /usr/bin/marco.bak,因为多数桌面会自动重启窗口管理器进程)，然后，就可以看到效果。

从分工来看，窗口Stack理应由窗口管理器负责，这也是窗口管理器的主要职责之一，所以，窗口Stack必然对窗口管理器可见，而且必定有接口来控制窗口的Z序。

### 查看Stack

那么如何查看窗口Stack呢？

通过上一节提到的XQueryTree接口，可以查询root窗口所有的children窗口列表，这个列表，本质上就是窗口Stack，只是顺序刚好想法，最后一个child位于列表的最后，所以，利用上一节的代码，即可查询到窗口Stack。

但是，其实，还有更简单的方法，有一套shell命令，可以查看窗口的各种详细信息，他们属于xorg-x11-utils软件包，安装即可使用。

这里用到的命令为xwininfo命令，通过如下命令，可用查询到详细的窗口stack信息(还包括父子关系)，示例如下：

    [root@localhost marco-1.8.2]# xwininfo -root -tree
    xwininfo: Window id: 0xae (the root window) (has no name)
      Root window id: 0xae (the root window) (has no name)
      Parent window id: 0x0 (none)
         97 children:
         0x2c00003 "Fcitx Input Window": ("fcitx" "fcitx")  600x61+1+333  +1+333
         ...
         0x3c00292 "终端": ("mate-terminal" "Mate-terminal")  187x220+287+314  +287+314
            1 child:
            0x3c00293 (has no name): ()  1x1+-1+-1  +286+313
         0x1a00d11 "Caja": ("caja" "Caja")  189x330+518+181  +518+181
            1 child:
            0x1a00d12 (has no name): ()  1x1+-1+-1  +517+180
         0x1a00c9e "Caja": ("caja" "Caja")  170x41+1367+769  +1367+769
         0x1600003 "下方扩展边缘面板": ("mate-panel" "Mate-panel")  1366x38+0+730  +0+730
            2 children:
            0x1600023 (has no name): ()  1366x38+0+0  +0+730
            ...

Stack中，包含unmaped和maped的所有窗口，unmaped的窗口对用户来说时不可见的，不会显示到桌面中;mapped的窗口，根据其叠放顺序显示。

### QT BypassWindowManagerHint|FramelessWindowHint

消失的QT界面设置了BypassWindowManagerHint属性，该属性设置的目的如下：

- 让该窗口××脱离××窗口管理器的管理，直观表现为：
  - panel中看不到相应的图标
  - Alt+Tab键切换最近使用窗口时，看不到相应的图标
- 让该窗口位于所有窗口的前面(Above)，即其它任何窗口都不能覆盖本窗口。

另外设置了FramelessWindowHint属性，目的是“不要边框”(默认情况下，窗口管理器会为所有的普通窗口加上边框，包括最大化、最小化、关闭按钮等)

这也是这两个属性的本意，悬浮菜单需要这样的效果。

QT 5.3中该属性设置，相关代码如下：

    enum WindowType {
            Widget = 0x00000000,
            Window = 0x00000001,
            ...
            BypassWindowManagerHint = 0x00000400,
            X11BypassWindowManagerHint = BypassWindowManagerHint,
            FramelessWindowHint = 0x00000800,
            ...
          }
设置Xorg相关属性：

    void QXcbWindow::setNetWmWindowFlags(Qt::WindowFlags flags)
    {
        // in order of decreasing priority
        QVector<uint> windowTypes;
        ...

        if (flags & Qt::FramelessWindowHint)
            windowTypes.append(atom(QXcbAtom::_KDE_NET_WM_WINDOW_TYPE_OVERRIDE));

        windowTypes.append(atom(QXcbAtom::_NET_WM_WINDOW_TYPE_NORMAL));
        ...
    }

可见，设置这两个属性后，相应的Xorg窗口type会设置为：

    _KDE_NET_WM_WINDOW_TYPE_OVERRIDE

该属性对于Xorg来说，就是脱离窗口管理器管理的，位于最前面的窗口，窗口管理器对于这里窗口会做特殊处理，相应类型也是某标准协议中写明的。

### Marco如何管理Stack

回到关键的Marco，关键的问题：Marco如何管理窗口Stack？

其管理机制非常简单：

将新窗口插入到除具有Override属性之外的所有的用户可见的窗口之前。

这里有几个关键点：

1. 具有Override属性的窗口，对于QT来说，就是设置了BypassWindowManagerHint|FramelessWindowHint的窗口。
2. 用户可见的窗口，即经过Map(XMapWindow)操作的窗口。
3. 新窗口，即新创建的窗口，这是从Marco的角度看的。通常来说应用程序使用XCreateWindow之类的Xlib接口创建窗口，这个操作由Xorg负责完成，Marco不参与，那Marco如何知道有新窗口创建呢？实际上，marco只监听从Xorg发送过来的事件，由专门的事件循环处理(event_callbak),用户创建窗口时不发送事件，但在Map窗口时，Xorg会向marco发送事件，此时marco会进行相应处理，管理Stack主要在这个流程中进行，后面单独介绍。

按此机制，可以保证最有的Override窗口一直保持在最前面，不会被覆盖。

Marco中调整Stack顺序的主要代码流程如下：

    event_callback
      meta_window_new  //创建Marco端的窗口数据结构
        meta_window_new_with_attrs
          meta_display_register_x_window //将新窗口加入到自己维护的hash表中，
                                           很关键，后面重点说明
          meta_window_reload_properties //设置相关窗口属性后，重新加载属性，
                                          如果相关属性被修改，会导致相
                                          关的callback被调用，后面会进行说明
          meta_window_ensure_frame //为窗口添加边框(创建父窗口包含它)
          meta_workspace_add_window //将新窗口加入工作区
          meta_stack_add //将新窗口加入自己的Stack中
            stack_sync_to_server //同步自己的Stack和Xorg的Stack
              raise_window_relative_to_managed_windows//会进行上述所说动作:
                                                        将新窗口插入插入合
                                                        适的位置

整个流程比较复杂，我们只关注关键的部分raise_window_relative_to_managed_windows函数进行了实际的Stack调整操作：

    static void
    raise_window_relative_to_managed_windows (MetaScreen *screen,
                                              Window      xwindow)
    {
      ...
      // 从Xorg获取其Stack信息
      XQueryTree (screen->display->xdisplay,
                  screen->xroot,
                  &ignored1, &ignored2, &children, &n_children);

      ...
      i = n_children - 1;
      // 从最后的child(最前端)开始，论询Stack，找到合适的未知
      while (i >= 0)
        {
          // 最后的child就是窗口本身，说明当前窗口已经位于最前面，需要继续轮询。
          if (children[i] == xwindow)
            {
              /* Do nothing. This means we're already the topmost managed
               * window, but it DOES NOT mean we are already just above
               * the topmost managed window. This is important because if
               * an override redirect window is up, and we map a new
               * managed window, the new window is probably above the old
               * popup by default, and we want to push it below that
               * popup. So keep looking for a sibling managed window
               * to be moved below.
               */
            }
          // 查找自己维护的hash表，该hash表中包含了所有的maped且非Override的窗口，该表的维护机制后面说明。如果找到，说明这个child就是我们插入的参考了，由于是从上到下依次轮询，最先查到的必然就是当前显示在最前面的窗口了。另一方面，由于Override窗口并未加入到hash表中，所以Override窗口永远不会作为参考，也就是说，新加入的窗口的插入位置永远不会超过Override窗口的位置。
          else if (meta_display_lookup_x_window (screen->display,
                                                 children[i]) != NULL)
            {
              XWindowChanges changes;

              /* children[i] is the topmost managed child */
              meta_topic (META_DEBUG_STACK,
                          "Moving 0x%lx above topmost managed child window 0x%lx\n",
                          xwindow, children[i]);

              changes.sibling = children[i];
              changes.stack_mode = Above;

              meta_error_trap_push (screen->display);
              // 向Xorg发送请求，调整Stack顺序，参考该child当前的位置，将新窗口放到该child前面，具体实现请看Xorg中的ConfigureWindow函数
              XConfigureWindow (screen->display->xdisplay,
                                xwindow,
                                CWSibling | CWStackMode,
                                &changes);
              meta_error_trap_pop (screen->display, FALSE);

              break;
            }

          --i;
        }
        ...
    }

### Marco窗口Hash表维护

前面提到，Marco自己维护的窗口Hash表对于Stack的维护至关重要，那这个Hash是如何维护的呢？

主要原则为：只有maped(对用户可见的)、而且是××非××Override窗口才能加入到该列表中。换句话说，窗口管理器只管理用户可见的需要自己管理的(设置BypassWindowManagerHint属性的窗口显然就是不需要被管理)窗口。相关主要代码流程如下：

    event_callback
      meta_window_new  //创建Marco端的窗口数据结构
        meta_window_new_with_attrs
          meta_display_register_x_window //将新窗口加入hash表，但有一定条件

在meta_window_new_with_attrs()函数流程中，进行了相应判断，对于Override窗口，和unmap(!IsViewable)窗口，不会走到meta_display_register_x_window流程，相关代码如下：

    if (attrs->override_redirect)
      {
        meta_verbose ("Deciding not to manage override_redirect window 0x%lx\n", xwindow);
        return NULL;
      }
      ...
      if (must_be_viewable && attrs->map_state != IsViewable)
      {
        ...
        return NULL;
      }
      ...

除了上述流程，还有例外情况，后面再单独说明。

### 看起来一切正常？

上诉的所有机制，凑一起，达到了基本目的：override窗口在最前面，其它新窗口依次排列。

看起来好像没有问题，理应运转正常，但问题确实出现了。

## 故障发生
### user_time & user_time_window
凡是总有例外，这个例外就是由于Xorg环境中加入的user_time属性，该属性的目的是增加桌面系统的易用性，假设这样一种场景：

    程序员正在终端或者IDE窗口中编写代码，突然来了个弹出窗口，
    此时弹出窗口默认会抢占焦点，导致编程工作被中断，是否会有不爽～

另一种场景：

    用户正在桌面中写文档，突然想打开浏览器看看邮件，而浏览器偏僻不争气，
    启动巨慢，在浏览器启动期间，用户可能不会傻等，时间宝贵，继续写文档，
    在写文档的过程中，浏览器打开了，此时浏览器默认会抢占焦点，而文档编
    写过程被打断，难得的灵感可能瞬间消失了，是否会有不爽～

上述两种典型的场景中，用户可能更期望的是：当前交互操作不被中断。而引入user_time属性正是为了解决这样的问题(Windows好像没有解决～)，实在用心良苦，但其实代价也不小。

新机制需要应用程序在每次有交互动作时(比如键盘或鼠标事件处理过程中)自己设置窗口的user_time属性，然后要求窗口管理器对窗口user_time属性进行判断和处理，以正确处理焦点问题。

窗口管理器处理理所当然，要求每个应用程序都修改进行适配，这个似乎有点过分，但这个已经写入相应的协议规范中了(估计确实没有其它更好的办法~)，很多应用程序也多照做了～，但是还有一个问题：

很多应用程序都会创建不只一个窗口，要求在每个窗口的交互事件中进行user_time处理似乎太过分了？

所以，在加入user_time属性的同时，加入了user_time_window属性，该属性标识处理user_time的窗口ID，即每个应用程序只需要设置一个user_time_window，然后仅对user_time_window处理user_time即可，这样，要求就降低了。

通常情况下，建议将user_time_window设置为toplevel窗口即可，但也有些应用程序使用了特别的其它的方式，比如Wine，后面说明。

### 注册user_time_window

窗口管理器需要处理user_time,那就必须要管理user_time_window，否则达不到预期效果。所以，当应用程序设置user_time_window时，Marco需要将相应的窗口加入到自己的hash表中，实现方法为：注册相应属性的修改通知事件，当相关属性修改时，会通知Marco进行处理，Marco在相应的回调函数中进行注册，具体代码为：

    static void
    reload_net_wm_user_time_window (MetaWindow    *window,
                                    MetaPropValue *value,
                                    gboolean       initial)
    {
      if (value->type != META_PROP_VALUE_INVALID)
        {
          /* Unregister old NET_WM_USER_TIME_WINDOW */
          if (window->user_time_window != None)
            {
              /* See the comment to the meta_display_register_x_window call below. */
              //把原来注册的窗口清理掉
              meta_display_unregister_x_window (window->display,
                                                window->user_time_window);
              /* Don't get events on not-managed windows */
              XSelectInput (window->display->xdisplay,
                            window->user_time_window,
                            NoEventMask);
            }


          /* Obtain the new NET_WM_USER_TIME_WINDOW and register it */
          window->user_time_window = value->v.xwindow;
          if (window->user_time_window != None)
            {
              /* Kind of a hack; display.c:event_callback() ignores events
               * for unknown windows.  We make window->user_time_window
               * known by registering it with window (despite the fact
               * that window->xwindow is already registered with window).
               * This basically means that property notifies to either the
               * window->user_time_window or window->xwindow will be
               * treated identically and will result in functions for
               * window being called to update it.  Maybe we should ignore
               * any property notifies to window->user_time_window other
               * than atom__NET_WM_USER_TIME ones, but I just don't care
               * and it's not specified in the spec anyway.
               */
              //注册新窗口
              meta_display_register_x_window (window->display,
                                              &window->user_time_window,
                                              window);
              ...
            }
        }
    }

代码足够简单。但这里有个问题，其加入hash表(注册)时，没有判断该窗口是否为maped，就是说unmaped的窗口也可能会被加入hash表中，看起来与该hash表设计的初衷不太相符，注册前的一大段注释也说明作者也有这方面的疑虑，但作者比较任性，don't care，因为没有明文规定。正是这里导致了我们遇到的问题。

### 问题来了

前面所说，由应用程序自己设置user_time_window，如果设置为toplevel window，一切OK，因为toplevel window必然会map。

但有些应用程序偏僻不这么干(具体原因没有去深究)，比如Wine，其创建一个1X1的假窗口来作为user_time_window，而且该窗口并未被map，相关代码流程如下：

    create_whole_window
      set_initial_wm_hints
        update_user_time //其中创建1x1窗口
        XChangeProperty // 设置为user_time_window

update_user_time()中创建1x1窗口：

    void update_user_time( Time time )
    {
        if (!user_time_window)
        {
            //创建1X1窗口
            Window win = XCreateWindow( gdi_display, root_window, -1, -1, 1, 1, 0, CopyFromParent,
                                        InputOnly, CopyFromParent, 0, NULL );
            if (InterlockedCompareExchangePointer( (void **)&user_time_window, (void *)win, 0 ))
                XDestroyWindow( gdi_display, win );
            TRACE( "user time window %lx\n", user_time_window );
        }

        if (!time) return;
        XLockDisplay( gdi_display );
        if (!last_user_time || (long)(time - last_user_time) > 0)
        {
            last_user_time = time;
            //修改user_time属性
            XChangeProperty( gdi_display, user_time_window, x11drv_atom(_NET_WM_USER_TIME),
                             XA_CARDINAL, 32, PropModeReplace, (unsigned char *)&time, 1 );
        }
        XUnlockDisplay( gdi_display );
    }

Marco在感知到_NET_WM_USER_TIME_WINDOW属性被修改后，会进入上述reload_net_wm_user_time_window流程，直接将该窗口加入hash表，而不顾该窗口是否为maped。

加入hash表后，问题就来了，如之前描述的marco维护Stack的机制，新窗口会默认插入能在hash表中找到的第一个child的前面，也就说后面的窗口会默认被插入到这个1X1窗口的前面，这样，Stack就乱了，无法保证Override窗口不被覆盖了。

试想，如果先打开了一个Override窗口，此时该窗口位于最前面，然后打开wine程序，此时1X1窗口和窗口位于最前面，然后再打开一个全屏窗口，该全屏窗口会被插入到1X1窗口前面，也就是最前面，其就覆盖掉了Override窗口。过程描述如下：

    1：open Override window，Stack如下：
        Override Win
        Normal Win
    2：open Wine，Stack如下：
        1X1 Win
        Override Win
        Normal Win
    3：open full-screen Window，Stack如下：
        full-screen Win
        1X1 Win
        Override Win
        Normal Win
Override窗口被覆盖了！

如果不是设置1X1窗口为user_time_window，这种情况下，Override窗口不会被覆盖(正常过程)，过程描述如下：

    1：open Override window，Stack如下：
        Override Win
        Normal Win
    2：open Wine，Wine窗口加入到Normal Win前面，Override Win后面。Stack如下：
        Override Win
        Wine Win
        Normal Win
    3：open full-screen Window，full-screen win加入到Wine win前面，Override Win后面。Stack如下：
        Override Win
        full-screen Win
        Wine Win
        Normal Win      
如此，Override窗口永远不会被覆盖。

# 问题总结

问题看似多方面的问题：

- 不用user_time就不会有问题
- 不设置unmaped的窗口为user_time_window也不会有问题
- marco能正确处理unmaped的窗口为user_time_window也不会有问题

# 解决

不能要求应用程序自身去解决或规避，最好还是在窗口管理器中处理，有两种处理思路：

1. 在Marco将user_time_window加入hash表时，判断窗口属性是否为maped，如果不是，则不加入。如此能解决stack混乱的问题，非maped的窗口不被管理。但问题是：类似应用程序的user_time属性就失去作用了，也就不能达到之前描述的预期目的。但，说实在话，该功能并不是必须的，对易用性的改进程度尚未可知，而且对性能也有一定的牺牲，所以，从某种程度上，也可以说：不用也罢！

2. 可用将这样的unmaped窗口加入到hash表中，但加入时，需要正确处理Stack顺序，至少需要保证加入的窗口位于所有的Override窗口之下，以保证其不被覆盖。但这个实现起来相对比较困难，改动更大，更复杂，风险更大，还需要考虑各种时序和同步问题，一不小心，可能将Stack弄得更乱。需慎重考虑进行。
