---
layout: post
title:  "gtkperf工具改造for GTK3+"
date:   2016-01-28 06:06:09
author: Jiang Biao
categories: Graphic
---

# 闲话

当前图形环境中，针对图形性能和用户体验方面的性能评估工具非常稀缺，关于这方面足够用单独的一篇文章来讨论，后面有机会再聊。

这里主要聊gtkperf工具， gtkperf工具是开源的用于测试GTK性能的工具， 确切的说应该是测试GTK中不同的控件的反应速度的工具。 对于Gnome的环境，由于大部分工具都是基于GTK开发的，所以gtkperf的测试结果能一定程度上反应图形环境的性能，当然这么多也很不准确，由于gtkperf工具只是简单的重复的绘制相同的空间，跟实际的用户操作和应用场景差别很多，业内很多人都认为这样工具纯属*扯淡*，但个人认为，其还是有一定的价值，至少在当前图形相关的benchmark工具严重稀缺的情况下。

gtkperf基本的工作原理为：重复的绘制指定的控件和图形元素，绘制次数可以选择，测试的控件也可以选择，包括一些基本的控件，比如：button、combobox、progress bar、entry、radioBox等，还包括一些基本的图形元素的绘制，比如：直线、圆形(并填充)、pixbuf等。其功能比较简单和单一，具体的原理和代码分析在单独的文章的再聊了。

# 现有gtkperf工具的问题

现有gtkperf工具在很多的发现版中都自带了，比如fedora的版本，使用也非常简单，指针执行gtkperf命令，在弹出的界面中操作即可看到结果，易用性很好。

但是，其有如下主要问题：

1. 基于GTK2+开发，只能用于测试GTK2+的性能，对于GTK3+的环境(比如Gnome3环境)无能为力，而现如今，GTK3+必然是GTK的未来和主流，不能支持GTK3+，就意味着*落伍*，就有被淘汰的可能。
2. 由于基于GTK2+开发，使用了一些与X11强绑定的接口，导致其无法直接用于Wayland后端的环境，从而无法用于测试Wayland环境的性能。
3. 处于**不维护**的状态，从gtkperf的官方git看，自2006年起，该工具就再也没有更新过，一方面可见其*年岁已长*，另一方面可见已无人维护。如果还支持新功能或者改造，只能自己动手了。
4. 其绘制图形时，部分操作的合理性有待商榷，比如其绘制直线时，是一条条单独绘制的，而实际的应用场景中，是很少这样使用的，多会利用批量处理接口来提高性能，也许也是因为其*年长*的原因导致。

# 针对GTK3+的移植

如前面所说，现有gtkperf工具无法支持GTK3+的测试，所以，对该工具进行了针对性的改造(GTK3+无法保持与GTK2+兼容，所以这个比较讨厌)，按Gnome官方提供的指南，一点点解决编译错误，直至改造成功。

细节这里就不说了，主要说明如下几点：

- Gnome提供了将GTK2+程序改造为GTK3+程序的指南，请参考：

[Migrating from GTK+ 2.x to GTK+ 3](https://developer.gnome.org/gtk3/stable/gtk-migrating-2-to-3.html)

[Changes that need to be done at the time of the switch](https://developer.gnome.org/gtk3/stable/ch25s02.html)

- 基于GTK3+的环境(主要是相关的依赖库)重新编译gtkperf工具，需要使用如下的编译选项：
	`pkg-config --cflags --libs gtk+-3.0`

	示例如下：

	gcc `pkg-config --cflags --libs gtk+-3.0` hello.c -o hello

移植后的代码，我后面再抽空放到我的github上。

