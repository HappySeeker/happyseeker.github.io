---
layout: post
title:  "Wayland & Input"
date:   2016-12-13 12:22:46
author: HappySeeker
categories: Graphic
---

前段时间分析Xorg中键盘相关问题，顺便分析了一下Wayland中Input相关部分，这篇文章做简单介绍，涉及内容比较多，可能有点乱，权当笔记。

# Wayland中的Input概况

与Xorg一样，Wayland中显然也需要统一的Input处理框架和机制，Wayland复用了现有的功能模块，省了不少力，主要结构如下图所示：

![Wayland Input架构图](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/wayland-input-arch.png)

从图中可以看出，libinput是wayland input模块的核心。

libinput是一套通用的用于处理Input设备的库，提供功能如：设备检测、设备处理、输入事件处理以及各种抽象，可让使用者在实现想要的Input相关功能的同时，使其定制化代码最小化。输入事件包括：触摸屏、鼠标和键盘等各种各样输入设备产生的事件。

libinput供display servers使用，wayland和Xorg都可以用，应用架构分别如下：

Wayland：

![libinput在Wayland中应用架构](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/libinput-for-wayland.png)

Xorg：

![libinput在Xorg中应用架构](https://github.com/HappySeeker/happyseeker.github.io/raw/master/_posts/libinput-for-xorg.png)

另外，libevdev是一套与内核中evdev驱动接口的用户态程序，对evdev设备的所有的ioctl操作进行了封装，方便用户使用，libinput中用到了libevdev相关接口处理事件，但是weston中没有使用。

libxcbcommon实现了XKB扩展协议中的相关标准接口，提供给应用程序使用，weston使用该库来处理XKB扩展协议相关的内容，比如capslock键对应的modifier的更新。

# Input事件处理基本原理

Input事件处理是这里所关注的关键，其基本原理简单描述为：

- 用户态libinput利用epoll机制监控相关设备文件(/dev/input/eventx)上的操作(输入)。
- 当有Input事件(比如键盘按键)发生时，事件依次通过内核中的input device driver和input core到达evdev driver，evdev driver中唤醒通过epoll睡眠在该设备文件上的进程，并将相应的事件数据转发到用户态。
- 用户态进程(wayland compositor/weston)被唤醒，接收相应事件数据，进行相关处理，并根据需要将其转发到wayland client程序。

看似简单，实际涉及非常复杂的细节，一些关键点：

- 内核中Input子系统对于Wayland和Xorg来说，是完全一样的，Wayland的input复用了现有的内核中的Input基础设施，不必重新开发。
- Wayland中利用了libinput和libxkbcommon通用库，实际也是一种复用。
- libinput利用了udev来监控来自内核的input设备事件，并通过读取input设备对应的udevnode(对应为/dev/input/eventx)来获取epoll监控的设备。复用了现有的udev相关机制，比较巧妙。
- libinput利用epoll来监控input设备上的事件(相应事件从内核中evdev驱动中转发上来)，由于epoll的高性能、高并发特性，使得libinput处理输入事件非常高效。同时复用了现有的epoll机制，不必另行设计实现。

可以看出，Wayland尽可能的复用了现有的机制，减少了开发工作量，使其专注于核心部分的开发，所以整体来看，Wayland无论从体量上，还是从架构上，都比Xorg轻量、简单许多。

# 流程 & 代码分析

## Input事件监控初始化

如前面所说，Wayland通过libinput处理Input事件，libinput通过epoll机制监控输入设备，下面分析主要相关代码：

Input事件监控初始化的主要代码流程如下：

    drm_compositor_create // 创建compositor
      udev_input_init // input初始化
        udev_input_enable
          udev_input_add_devices
            device_added // 从input设备对应的uevent中读取udevnode，并将其加入到epoll监控中
              evdev_device_create
                wl_event_loop_add_fd
                  add_source // 实际将input设备对应的fd加入到epoll监控中

evdev_device_create函数关键代码如下：

    struct evdev_device *
    evdev_device_create(struct weston_seat *seat, const char *path, int device_fd)
    {
      ...
      // device_fd是通过udev从相应设备中读取的，对应于/dev/input/eventx设备的fd
      device->fd = device_fd;
      ...
      // 获取设备名称
    	ioctl(device->fd, EVIOCGNAME(sizeof(devname)), devname);
    	devname[sizeof(devname) - 1] = '\0';
    	device->devname = strdup(devname);

    	if (evdev_configure_device(device) == -1)
    		goto err;

    	if (device->seat_caps == 0) {
    		evdev_device_destroy(device);
    		return EVDEV_UNHANDLED_DEVICE;
    	}
      //创建dispatch，后续在事件循环中使用
    	/* If the dispatch was not set up use the fallback. */
    	if (device->dispatch == NULL)
    		device->dispatch = fallback_dispatch_create();
    	if (device->dispatch == NULL)
    		goto err;
      //将input设备对应的fd加入到epoll监控中，并设置事件发生时的处理接口为evdev_device_data(后面说明)
    	device->source = wl_event_loop_add_fd(ec->input_loop, device->fd,
    					      WL_EVENT_READABLE,
    					      evdev_device_data, device);
    	...
    }

add_source函数主要将input设备对应的fd加入到epoll监控中，关键代码如下：

    static struct wl_event_source *
    add_source(struct wl_event_loop *loop,
    	   struct wl_event_source *source, uint32_t mask, void *data)
    {
      struct epoll_event ep;
      ...
    	// 初始化事件类型，前面传入的mask为WL_EVENT_READABLE，所以这里设置的event类型应该时EPOLLIN，即当设备上有“输入”事件时，唤醒处理
    	memset(&ep, 0, sizeof ep);
    	if (mask & WL_EVENT_READABLE)
    		ep.events |= EPOLLIN;
    	if (mask & WL_EVENT_WRITABLE)
    		ep.events |= EPOLLOUT;
    	ep.data.ptr = source;
      // 通过epoll_ctl接口将input设备对应的fd加入到监控列表中
    	if (epoll_ctl(loop->epoll_fd, EPOLL_CTL_ADD, source->fd, &ep) < 0) {
    		close(source->fd);
    		free(source);
    		return NULL;
    	}
      ...
    }

## 事件处理

事件处理从epoll检测到相应的“输入”事件开始，如前面分析，在evdev_device_create函数中设置了相应的回调evdev_device_data，所有事件处理就从evdev_device_data函数开始，其主要代码流程如下(以键盘capslock按键事件为例)：

    evdev_device_data // 入口
      evdev_process_events // 调用前面evdev_device_create中创建的dispatch中的相应接口，设置为fallback_process
        fallback_process // 根据事件类型(比如，鼠标移动、键盘按键)选择相应的事件处理，键盘按键走evdev_process_key
          evdev_process_key
            notify_key // 通知client按键事件
              update_modifier_state // 更新modifier状态，capslock需要更新相应的lock modifier
                xkb_state_update_key // 使用libxkbcommon接口更新相应的modifier
                  xkb_filter_apply_all

其中，dispatch默认使用如下接口：

    struct evdev_dispatch_interface fallback_interface = {
      fallback_process,
      fallback_destroy
    };

## Capslock对应modifier更新

分析Capslock键按下后相应的modifier是如何更新的，主要关注大小写锁的**生效时机**，即在按键按下时生效，还是按键弹起后生效。

经实测，wayland中的capslock键与Xorg环境中存在相同的问题：当快速输入时，会出现大小写混乱问题，具体表现为：

在当前状态为**大写**状态时，当capslock按下后、弹起之前，有字符按键按下时，结果为大写，而此时用户期望其为小写。

如下分析相关原因(其本质上与Xorg中一样，实现原理也类似)，相关逻辑主要集中在xkb_filter_apply_all函数中，这段代码不太好理解，先看看关键代码：

    /**
     * Applies any relevant filters to the key, first from the list of filters
     * that are currently active, then if no filter has claimed the key, possibly
     * apply a new filter from the key action.
     */
    static void
    xkb_filter_apply_all(struct xkb_state *state,
                         const struct xkb_key *key,
                         enum xkb_key_direction direction)
    {
        // 抽象了一个filter概念，其中包含关键接口func，用于实际执行filter操作，即修改modifier。
        struct xkb_filter *filter;
        const union xkb_action *action;
        bool send = true;

        /* First run through all the currently active filters and see if any of
         * them have claimed this event. */
        /* 遍历现有的所有filter确认其func是否已经初始化，需要注意：filter的func是在后面
         * 的代码中初始化的，实际是在按键(如capslock)按下之后，即在按键按下之后这里代码不会
         * 执行，只有在按键弹起后才会执行到这里。
         */
        darray_foreach(filter, state->filters) {
            if (!filter->func)
                continue;
            send = filter->func(state, filter, key, direction) && send;
        }
        /* 判断当前按键状态，XKB_KEY_UP表示按键弹起，即按键弹起时，不进行后面的操作，仅执行
         * 前面的filter->func; 当按键按下时执行后面的流程
         */
        if (!send || direction == XKB_KEY_UP)
            return;
        // Xkb扩展协议中规范了每个按键事件的执行流程，其中一个步骤就是执行每个按键对应的action
        action = xkb_key_get_action(state, key);

        // 这里开始初始化filter
        filter = xkb_filter_new(state);
        if (!filter)
            return; /* WSGO */

        filter->key = key;
        // 设置filter->func，从filter_action_funcs中读取，该接口中实际实现了lock modifier修改。后面单独分析
        filter->func = filter_action_funcs[action->type].func;
        filter->action = *action;
        // 调用filter_action_funcs中定义的new接口，其中设置了setmods，后面单独分析
        filter_action_funcs[action->type].new(state, filter);
    }

其中的几个关键点：

- 通过按键状态控制执行的操作，当按键按下时，执行 filter_action_funcs[action->type].new，并初始化filter->func；当按键释放时，执行filter->func操作。
- 通过filter_action_funcs列表来控制实际执行的动作，filter_action_funcs是一个数组，每个数组元素包含了两个函数指针，执行具体的操作，具体定义如下：

      static const struct {
          void (*new)(struct xkb_state *state, struct xkb_filter *filter);
          bool (*func)(struct xkb_state *state, struct xkb_filter *filter,
                       const struct xkb_key *key, enum xkb_key_direction direction);
      } filter_action_funcs[_ACTION_TYPE_NUM_ENTRIES] = {
          [ACTION_TYPE_MOD_SET]    = { xkb_filter_mod_set_new,
                                       xkb_filter_mod_set_func },
          [ACTION_TYPE_MOD_LATCH]  = { xkb_filter_mod_latch_new,
                                       xkb_filter_mod_latch_func },
          [ACTION_TYPE_MOD_LOCK]   = { xkb_filter_mod_lock_new,
                                       xkb_filter_mod_lock_func },
          [ACTION_TYPE_GROUP_SET]  = { xkb_filter_group_set_new,
                                       xkb_filter_group_set_func },
          [ACTION_TYPE_GROUP_LOCK] = { xkb_filter_group_lock_new,
                                       xkb_filter_group_lock_func },
      };

两个函数指针分别为new和func，这里颇有函数式编程的味道，将函数作为对象看待，大家可以体会下。

capslock键对应的元素为ACTION_TYPE_MOD_LOCK，其对应的函数指针分别为xkb_filter_mod_lock_new和xkb_filter_mod_lock_func，也就是说：

当capslock按下时，执行xkb_filter_mod_lock_new函数，当释放时执行xkb_filter_mod_lock_func函数。

xkb_filter_mod_lock_new函数的代码如下：

    static void
    xkb_filter_mod_lock_new(struct xkb_state *state, struct xkb_filter *filter)
    {
        filter->priv = (state->components.locked_mods &
                        filter->action.mods.mods.mask);
        // 设置set_mods，在上级函数xkb_state_update_key中会根据该标记，设置base modifier
        state->set_mods |= filter->action.mods.mods.mask;
        if (!(filter->action.mods.flags & ACTION_LOCK_NO_LOCK))
            // 设置lock modifier
            state->components.locked_mods |= filter->action.mods.mods.mask;
    }

可以看出，当capslock键按下时，会设置base modifier和lock modifier，设置后，当前的状态为**大写**。即当capslock键按下后、释放前，永远是**大写**。

xkb_filter_mod_lock_func函数的关键代码如下：

    static bool
    xkb_filter_mod_lock_func(struct xkb_state *state,
                             struct xkb_filter *filter,
                             const struct xkb_key *key,
                             enum xkb_key_direction direction)
    {
        ...
        // 设置clear_mods，其上级函数xkb_state_update_key中会根据该标记，清理base modifier
        state->clear_mods |= filter->action.mods.mods.mask;
        if (!(filter->action.mods.flags & ACTION_LOCK_NO_UNLOCK))
            // 根据之前的状态设置locked_mods
            state->components.locked_mods &= ~filter->priv;
        // 按键释放后清除filter->func
        filter->func = NULL;
        return true;
    }

可以看出，当capslock键释放后，会清除base modifier，并根据之前的状态设置locked_mods，也就是说仅当按键**释放**后，capslock的真实状态才会生效。

从上述的逻辑可以看出，wayland中对capslock的处理逻辑实际与Xorg中完全一致(只是变了个花样)，都是在按键按下时设置base modifier和lock modifier，在按键释放时清除base modifier，并根据之前的状态设置locked_mods，所以，Wayland中存在与Xorg中相同的问题。修正思路也一致，但是会导致与Xkb扩展协议的不兼容。

## 内核与用户态间的事件转发 & epoll

最后，再来理一下**内核事件是如何转发到用户态的？**

前面好像已经说了，通过epoll？是的，但是这里面还有一个主要疑问：

- 前一篇分析Xorg中大小写混乱问题的文章中写得很清楚：内核中evdev driver是通过SIGIO将事件通知到用户态的，既然对于wayland来说内核中的逻辑完全一致，现在为什么又变成epoll了呢？纵观wayland的代码，也没有找到处理SIGIO信号的任何逻辑，难道epoll和SIGIO还有关系？

这个问题也是我之前没想明白的，在分析了epoll相关原理后才最后弄明白。

### epoll基本原理

我们知道，epoll是一种高效的、能提供高并发(轻松达百万数量级) 的异步IO方式，原有的老旧的poll/select机制性能不行，主要原因在于：其每次调用都需要将fd从用户态拷贝到内核态，当数量上去后，就会成为沉重的负担。当然还有其它原因，epoll的实现中有很多高效的设计，这里不一一描述。

epoll解决了主要的性能问题，提供3个主要的接口：

- epoll_create/epoll_ceate1。创建epollevent fs中的文件。epoll通过一个虚拟文件系统epollevent fs实现，具体原因和实现请搜搜。
- epoll_ctl。将需要监控的fd加入(或修改/删除)到监控列表中
- epoll_wait。等待被监控的fd上的事件发生

其解决关键问题(fd的拷贝)的核心在于epoll_ctl，在epoll_ctl后，内核中就创建了相应的结构，这样内核就已经知道了，所以后面就不用再拷贝了。其实际实现是还使用了其它的技术，比如缓存、红黑树等提高查找效率，效果很好，非常高效。

epoll_wait中，会将当前进程加入到fd对应的文件的wait队列中。这里是关键。需要了解：每个文件(包括socket和特殊文件)都一个wait队列，用于存放等待该文件事件(读写)的进程。epoll_wait后，如果没有事件发生，就会将当前进程加入wait队列，并注册相应的处理钩子ep_poll_callback。

当被监控的文件上有事件(读写)发生时(通常由中断触发)，会进入相应文件的处理流程中，在相应流程会处理事件，并**唤醒**wait队列，此后就会进入ep_poll_callback处理，ep_poll_callback中最终唤醒调用epoll_wait而进入等待队列的进程。这里有个关键点就是：**唤醒**wait队列的操作需要主动进行，就是说需要在特定的文件操作过程中主动调用，否则这些wait队列上的进程就不会唤醒了。搜搜内核代码，可以看到，socket、pipe等文件的处理过程中都会主动做唤醒操作，说明对于socket、pipe等文件的epoll操作是可以被唤醒的。

### 事件如何转发？

还是回到原本的问题，事件到底是如何转发的。

从上述的epoll基本原理可以看到，要将Input事件转发到用户态，其实就只需要一个简单的操作：唤醒相应设备文件对应的wait队列，剩下的工作都交给epoll了，但关键是这个**唤醒**操作需要主动手动调用。

再看看evdev driver的代码，其关键的转发流程在evdev_pass_values函数中：

    static void evdev_pass_values(struct evdev_client *client,
    			const struct input_value *vals, unsigned int count,
    			ktime_t *ev_time)
    {
      ...
    	for (v = vals; v != vals + count; v++) {
    		event.type = v->type;
    		event.code = v->code;
    		event.value = v->value;
        // 其中调用kill_fasync，向用户态进程发送SIGIO信号，其它的文章中已经分析过了
    		__pass_event(client, &event);
    		if (v->type == EV_SYN && v->code == SYN_REPORT)
    			wakeup = true;
    	}
      ...
    	if (wakeup)
        // 唤醒wait队列
    		wake_up_interruptible(&evdev->wait);
    }

可以看到，其中除了之前分析过的SIGIO的逻辑外(如果用户态没有注册SIGIO信号，那就算了～)，最后还唤醒了wait队列，于是epoll就能感知到了，于是事件就此转发到了用户态。

也就是说，SIGIO和epoll其实没啥关系，但可以并存，也没啥冲突。
