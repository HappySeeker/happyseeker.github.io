---
layout: post
title:  "Input子系统 & 快速输入时大小写混乱问题"
date:   2016-12-08 16:35:09
author: HappySeeker
categories: Graphic
---

# 问题

在标准Fedora系统中，登录系统输入密码时，当使用Capslock键切换大小写，且输入速度比较快时，会出现大小写混乱问题。

# 分析

## 确认现象

上述问题描述比较模糊，需要先自己做些深入测试验证，确认清楚现象，为后续分析指导方向。

具体现象：

1. 当前处于大写状态时(Capslock被按下过1次)，使用Capslock切换大小写，并快速输入密码时(在Capslock键按下后、弹起前，同时按下任意字母键)，用户本来期望其为小写，但实际为大写。当Capslock键弹起后，再次按任意字母键，均正常(小写)。即故障仅发生在“Capslock键按下后、弹起前，同时按下任意字母键”的情况下。
2. 当前处于小写状态时(Capslock未被按下)，按上述操作没有问题。即故障仅发生在“当前处于大写状态时，Capslock键按下后、弹起前，同时按下任意字母键”的情况。
3. 在图形环境中任意文本编辑类应用中(比如文本编辑器、终端、笔记软件)上都有相同的现象。说明故障应该跟具体应用程序无关，应在于Xorg或者操作系统本身(比如驱动)。
4. 切换到文本控制台，做同样操作，没有问题，说明故障应该与图形环境有关，结合前面的现象，基本可以推断故障在于Xorg。

## 相关架构和原理

问题在于对键盘按键的处理，键盘事件处理相关逻辑对应内核来说，属于Input子系统。对Input子系统的架构进行简化后的效果图如下所示：

![Input子系统架构](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/Input arch.png)

可见，Input子系统分为3个层次：

- Input device driver:与具体硬件(鼠标、键盘等)相关的驱动，其中包括相应的中断处理服务程序，通常只是将相应的事件向上层转发。
- Input Core：公共核心功能实现，包括核心数据结构和接口的定义。
- Input Event driver：事件处理驱动，对相应的Input事件进行实际处理。分两种情况：
  - 自己处理，比如，对于VT(虚拟终端)，实现了相应的Input Event driver，实现对键盘事件的处理。
  - 进一步转发。比如，evdev驱动，该驱动是标准的通用的事件处理驱动，图形环境中默认使用该驱动，其作用就是将事件转发到用户态程序，比如Xorg。

### 事件转发流程(以键盘按键为例)：

- 当用户按下一个按键时，硬件触发中断。
- 中断由Input device driver中的ISR进行处理。
- ISR将事件进一步向上层转发
- 不同的Input Event driver对事件进行处理。对于图形界面场景来说，evdev驱动将事件转发到用户态。
- Xorg监听到相应事件，进行处理。

### SIGIO相关

有个关键环节：evdev是如何将事件转发给Xorg的呢？

使用SIGIO，是Linux(应该说是类Unix系统)提供的一种异步IO机制，也称为信号驱动IO。

SIGIO本质上只是一种信号，编号为29(类比：Kill信号的编号为9),其专门用来实现异步IO。其用途类似于select(或poll)，用于异步通知IO事件，但与select又有不同，select需要进程主动去查询相应事件;而SIGIO不需要。

SIGIO具体实现涉及比较多的内容，这里就不详细说明了，有机会单独写文章来分析。这里简单描述其基本原理：

用户态进程注册对指定文件监听(open，然后fcntl设置FASYNC标志)，当指定文件上有IO操作时，内核向注册过的进程发送SIGIO信号。如此，用户态进程处理相应信号，然后感知到相应的事件。

evdev正是使用了该机制，用于将内核中的事件转发给Xorg和其它监听了Input事件的用户态进程。相应的主要代码流程为：

    evdev_event //evdev驱动注册的event接口
      evdev_events
        evdev_pass_values
          __pass_event // 转发事件
            kill_fasync
              kill_fasync_rcu
                send_sigio  // 发送信号

## Xorg相关分析

### SIGIO事件注册

接上述的SIGIO逻辑，Xorg部分需要注册相应的监听，主要代码流程为：

    EvdevKbdEnable
      KdRegisterFd
        KdAddFd
          KdNonBlockFd

其中，KdAddFd注册了SIGIO的信号处理接口。

    static void
    KdAddFd(int fd)
    {
        struct sigaction act;
        sigset_t set;

        kdnFds++;
        fcntl(fd, F_SETOWN, getpid());
        // 注册监听
        KdNonBlockFd(fd);
        AddEnabledDevice(fd);
        memset(&act, '\0', sizeof act);
        // 信号handler为KdSigio，evdev转发事件后，由此接口进行处理。
        act.sa_handler = KdSigio;
        sigemptyset(&act.sa_mask);
        sigaddset(&act.sa_mask, SIGIO);
        sigaddset(&act.sa_mask, SIGALRM);
        sigaddset(&act.sa_mask, SIGVTALRM);
        //注册信号
        sigaction(SIGIO, &act, 0);
        sigemptyset(&set);
        sigprocmask(SIG_SETMASK, &set, 0);
    }

KdNonBlockFd中注册监听，实现如下：

    KdNonBlockFd(int fd)
    {
        int flags;
        // 设置FASYNC标记，内核发送事件时，会依据此标记和相应的fd找到目标进程
        flags = fcntl(fd, F_GETFL);
        flags |= FASYNC | NOBLOCK;
        fcntl(fd, F_SETFL, flags);
    }

### Xorg事件处理

整体流程分两部分：

- 事件进一步转发
- 事件处理

事件进一步转发流程如下

    KdSigio //SIGIO信号的处理函数
      EvdevKbdRead //evdev中注册的键盘事件处理函数
        KdEnqueueKeyboardEvent // 将事件放入miEventQueue队列中
          QueueKeyboardEvents
            queueEventList
              mieqEnqueue

具体说明如下：

evdev转发的事件到达Xorg后，Xorg需要进行相应处理，如前面代码所示，处理入口为之前注册的SIGIO信号的处理函数：KdSigio

    static void
    KdSigio(int sig)
    {
        int i;

        for (i = 0; i < kdNumInputFds; i++)
            // 直接调用之前注册的read接口，对应键盘来说，注册的接口为：EvdevKbdRead
            (*kdInputFds[i].read) (kdInputFds[i].fd, kdInputFds[i].closure);
    }

EvdevKbdRead为之前evdev初始化时注册的接口，主要是将相应事件放入Xorg的输入事件队列，待Xorg的主循环处理

事件处理：

事件处理在Xorg的主循环中进行，即在Dispatch()函数中处理。在Dispatch中循环等待事件，主要有两种事件：Input事件和Client的请求，这里我们只关注Input事件，Input事件的处理在Client请求之前(表明Input的事件优先级相对较高)。主要的代码流程如下：

    Dispatch
      ProcessInputEvents
        mieqProcessDeviceEvent
          ProcessKeyboardEvent
            AccessXFilterPressEvent
              XkbProcessKeyboardEvent
                XkbHandleActions
                  _XkbFilterLockState//最终处理大小写问题的地方，后面详述

### XKB扩展协议

Xorg中键盘相关处理逻辑由XKB模块负责，其主要实现了XKB Extension(X Keyboard Extension)协议，该协议扩展了X协议中原有的针对键盘的相关标准，X协议中仅针对shift、control和Lock等键进行了基本行为的约定，但过于简单、有诸多限制，无法满足多样化的键盘和按键组合行为规范需求，于是有了该扩展协议，扩展协议定义了非常多的内容，比如键盘状态(Modifier)、全局键盘控制、键盘布局、键盘Bell、键盘指示器(如led等)等，这里仅关注键盘状态部分。

#### 键盘状态

核心是Modifier概念，Modifier用于体现modifier keys的状态，modifier keys是一些特殊功能的按键，比如Capslock、shift、control等，其定义类似于ISO9995 standard中的qualifier keys，标准定义为：

    A key whose operation has no immediate effect, but which, for as long as it is held down, modifies the effect of other keys. A qualifier key may be, for example, a shift key or a control key.

当某modifier key被按下或释放时，相应的状态(modifier)会随之改变。

##### Locked & Latched Modifiers

X协议中，无法区分一个modifier的设置是由于Lock键被按，或者是由于某其它键被按下未释放导致的。比如说：capslock键按后会设置相应的modifier，而shift键按下不释放也会设置相应的modifier(释放后reset)。

XKB扩展协议中，明确区分了这两种情况，将其分为Locked modifier和Latched Modifier，前者对应于Capslock按后的情况，后者对应于shift键按后的情况。Locked modifier在设置后一直生效，直到相应按键被再次按下；Latched modifier在设置后仅对于下一个按键生效。

除上述两个modifier外，XKB扩展协议中还定义了与此相关的另外两个modifier用于表明lock的状态，分别为：

- base modifier。Latched modifier和locked modifier通常反映物理按键的状态，而base modifier反映逻辑状态，其实就是另一种状态，提供给实现者使用，以区分更复杂的场景。
- effective modifier。就是前面3中modifer的累积和，实际实现为前面3中modifier做按位或操作结果，用于体现最终的大小写效果。

##### 键盘事件处理

XKB扩展协议中对键盘事件的处理过程进行了规范，分3个步骤：

1. 全局键盘控制。即一些全局的控制，用于决定“立即处理”、“延迟处理” or “忽略”。比如SlowKeys属性将导致键盘事件延迟、RepeatKeys属性将导致单个按键触发多个事件。

2. per-key行为。用于模拟或触发针对具体按键的某种特殊行为，比如不同的键盘布局下，相同按键将产生不同的键码。

3. key action。逻辑上，每一个键码都有其对应的action。对modifier的操作就在这个步骤中进行。

这里仅关注上诉步骤3,XKB扩展协议中对齐有更细化的说明，其中说明了XKB扩展协议支持如下类型的action：(仅列我们关注的几个，而且仅截取了部分关键内容，其它的请查看协议文档)

| Action    | Effect   |
| --------- | -------- |
| SA_NoAction | No direct effect, though SA_NoAction events may change the effect of other server actions (see below). |
| SA_SetMods | Key press adds any action modifiers to the keyboard’s base modifiers. ...|
| SA_LatchMods | Key press and release events have the same effect as for SA_SetMods ... Finally, key release latches any action modifiers that were not used by the clearLocks or latchToLock flags. |
| SA_LockMods | Key press sets the base and possibly the locked state of any action modifiers. ... For key release events, clears any action modifiers in the keyboard’s base modifiers ... |

重点关注SA_LockMods，因为Capslock键对应于这种类型的action，其意思是在按键按下时，设置base modifier和locked modifer，在按键释放时，恢复base modifier的值，但不恢复locked modifier，由此达到使大写lock生效的目的；对比下SA_LatchMods，其行为为在按键释放时会恢复action modifier(即Latched modifier)。

### Xorg中XKB相关实现

XKB中对于大小写的处理，关键就在于对于上述几个modifier的处理，相关处理逻辑在_XkbFilterLockState函数中，该函数在XkbHandleActions函数中调用，对应于上述“键盘事件处理步骤中的步骤3”。

XkbHandleActions函数的关键部分如下：

    void
    XkbHandleActions(DeviceIntPtr dev, DeviceIntPtr kbd, DeviceEvent *event)
    {
        ...
            if (sendEvent) {
                // 根据action的类型来选择相应的处理
                switch (act.type) {BN
                case XkbSA_SetMods:
                case XkbSA_SetGroup:
                    filter = _XkbNextFreeFilter(xkbi);
                    sendEvent = _XkbFilterSetState(xkbi, filter, key, &act);
                    break;
                case XkbSA_LatchMods:
                case XkbSA_LatchGroup:
                    filter = _XkbNextFreeFilter(xkbi);
                    sendEvent = _XkbFilterLatchState(xkbi, filter, key, &act);
                    break;
                // capslock键进入这里
                case XkbSA_LockMods:
                case XkbSA_LockGroup:
                    filter = _XkbNextFreeFilter(xkbi);
                    // 其中设置locked modifier
                    sendEvent = _XkbFilterLockState(xkbi, filter, key, &act);
                    break;
                ...
        // 根据_XkbFilterLockState中设置的标记，设置或者清除base modifier
        if (xkbi->setMods) {
            for (i = 0, bit = 1; xkbi->setMods; i++, bit <<= 1) {
                if (xkbi->setMods & bit) {
                    keyc->modifierKeyCount[i]++;
                    xkbi->state.base_mods |= bit;
                    xkbi->setMods &= ~bit;
                }
            }
        }
        if (xkbi->clearMods) {
            for (i = 0, bit = 1; xkbi->clearMods; i++, bit <<= 1) {
                if (xkbi->clearMods & bit) {
                    keyc->modifierKeyCount[i]--;
                    if (keyc->modifierKeyCount[i] <= 0) {
                        xkbi->state.base_mods &= ~bit;
                        keyc->modifierKeyCount[i] = 0;
                    }
                    xkbi->clearMods &= ~bit;
                }
            }
        }
        ...
      }

_XkbFilterLockState函数实现如下：

    static int
    _XkbFilterLockState(XkbSrvInfoPtr xkbi,
                        XkbFilterPtr filter, unsigned keycode, XkbAction *pAction)
    {
        ...
        // 按键按下时走这里
        if (filter->keycode == 0) { /* initial press */
            filter->keycode = keycode;
            filter->active = 1;
            filter->filterOthers = 0;
            filter->priv = xkbi->state.locked_mods & pAction->mods.mask;
            filter->filter = _XkbFilterLockState;
            filter->upAction = *pAction;
            if (!(filter->upAction.mods.flags & XkbSA_LockNoLock))
                // 这里设置locked modifier，即在按下时设置locked modifier
                xkbi->state.locked_mods |= pAction->mods.mask;
            xkbi->setMods = pAction->mods.mask;
        }
        // 按键释放时走这里
        else if (filter->keycode == keycode) {
            filter->active = 0;
            xkbi->clearMods = filter->upAction.mods.mask;
            if (!(filter->upAction.mods.flags & XkbSA_LockNoUnlock))
                // 释放时清除locked modifier
                xkbi->state.locked_mods &= ~filter->priv;
        }
        return 1;
    }

上述流程说明：当capslock被按下时locked modifier会被设置，意味者此时大小写状态一定是**大写**;只有在按键释放时才会清除locked modifier，意味着大小写最终状态一定是在按键释放后才会生效的，也就是说：
**当capslock键按下还未释放时，同时按下另外的字符按键，那么此时一定是大写的。小写仅可能出现在capslock按键释放之后**

这个现象跟我们之前测试现象是完全一致的。不符合用户的预期，于是问题出现了。

# 解决

XKB扩展协议中说明了SA_LockMods类型的action的具体行为，XKB模块中确实也照此实现了，如果要修改，相当于就不符合XKB协议了。

如果需要解决，那么需要修改该action的处理，实际上，我们需要的是当capslock按键按下后大小写状态转换即立刻生效，而不是在按键释放后才生效。所以，可行的解决思路如：在按键按下的流程中检测当前modifier的状态，将其toggle(翻转)，在按键释放时啥也不干。看起来要比原来的思路简单许多，但现实情况可能要复杂许多，XKB扩展协议中有许许多多与此相关的内容，如此修改可能会带来全局的影响，隐患很大。

风险更小的思路是，在按键按下并设置locked modifier后，将其清除一下，并且去掉设置base modifier的动作。风险相对更小，但由于没有完全吃透XKB协议，其它风险还未知，毕竟仅涉及Lock键的处理，估计影响不会太大。
