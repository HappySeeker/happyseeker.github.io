---
layout: post
title:  "闲聊Cairo"
date:   2016-02-01 07:09:30
author: Jiang Biao
categories: Graphic
---

# 闲话

今早还有些时间。 简单闲聊下Cairo。

初次见到这个名称时，会让人很迷惑，图形环境中术语实在太多了，让人迷乱，而这个名称本身也没有任何提示信息。

其实，Cairo在目前的桌面环境中，占有非常重要的地位。

# 什么是Cairo？

**Cairo**是什么？ 用来干嘛？ 如何用？ 

**Cairo**是一套用于绘制**2D** **矢量图形**库。 

首先，Cairo是用来绘制平台图形的，比如直线、矩形、圆、文字等，所以是**2D**，我们知道3D图形的绘制主要依赖于OpenGL(Linux中Mesa，Window中DirectX)提供的LibGL接口。

然后，再聊聊矢量图形。

# 什么是矢量图形？

概念上，计算机图形分两种：光栅图形(Raster graphics)和矢量图形(Vector graphics)。 

- 光栅图形用像素来表示，完全基于像素来描述，即由一个像素点来描述一张图像。
- 矢量图形则基于**几何元素**来表示。 这些**几何元素**包括：点、线、曲线、多边形，通过数学公式计算而成。

两种图形各有优缺点，矢量图形相对于光栅图形有如下优点：
1. 尺寸更小
2. 可无限放大，且不会影响图形的质量。 (因为放大后的图形是基于数学公式计算而成，不是通过像素级别的放大。)
3. 进行平移、填充和旋转操作不会影响图形的质量。

# Cairo有何特点？

我们知道，Cairo的应用非常广泛，最典型的就是在GTK+中的应用(其他应用如firefox)，尤其是GTK3+更是完全依赖于Cairo来绘制图形。 为什么大家会选择Cairo？ GTK+原来是基于Xlib直接绘制的(准确的说应该是GTK+基于GDK，而GDK基于Xlib绘制)，为什么会切换至Cairo呢？

Cairo针对不同的**后端**，向用户提供了一套统一编程接口。 Cairo的功能非常强大，几何能通过它绘制所有的平面图形。

**后端**(Backend)即底层的绘图引擎，Cairo向上层(用户)提供编程接口，对齐屏蔽底层的绘图细节，而具体绘图操作可以有很多种方式实现，比如我们最常见的Xlib的后端，就是通过Xlib的接口，向X Server(Xorg)端发送绘图请求，最终由Xorg完成图形绘制；再如OpenGL的后端，即通过OpenGL的接口(LibGL)，提供绘图需要的新，并向GPU发送绘制请求，由GPU完成图形绘制。

Cairo针对现有的主流的各种不同的后端，提供了一致的编程接口。 如此一来，上层用户即可以使用统一的接口，灵活使用不同的后端，实现图形绘制，大大增加了应用程序的可移植性、通用性和灵活性。 

对于GTK来说，原本只能使用Xlib(或Xcb)接口进行图形绘制，而如今，GTK3+完全基于Cairo后，理论上就可以使用不同的后端来绘制图形了。之所以说是**理论上**，是因为虽然GTK3+已经完全基于Cairo来绘制图形，但是其创建surface时，仍然写死指定了Xlib的后端所以，目前为止，GTK仍然只能使用Xlib来绘图，想要让GTK默认使用其它后端的同学们，恐怕要想其它办法了，而且，目前好像也没有看到Gnome官方有支持其它后端的计划(比如目前大热的OpenGL后端，QT都已经是默认使用了)，这个方向待深入研究再聊。

# Cairo支持什么后端?

Cairo支持的后端比较多，包括(这里只简单解释个人比较关注的后端)：

	typedef enum _cairo_surface_type {
	    CAIRO_SURFACE_TYPE_IMAGE,
	    CAIRO_SURFACE_TYPE_PDF, // pdf格式
	    CAIRO_SURFACE_TYPE_PS,
	    CAIRO_SURFACE_TYPE_XLIB, // Xorg中默认
	    CAIRO_SURFACE_TYPE_XCB,  // Xorg中使用，与Xorg采用异步通信，相比Xlib，效率稍高
	    CAIRO_SURFACE_TYPE_GLITZ,  // 老版本的cairo中使用的是Glitz库来使用OpenGL，Glitz库目前已经不维护，新版本中直接使用GL后端
	    CAIRO_SURFACE_TYPE_QUARTZ,  // Mac系统中使用的后端
	    CAIRO_SURFACE_TYPE_WIN32,  // windows系统中使用的后端
	    CAIRO_SURFACE_TYPE_BEOS,
	    CAIRO_SURFACE_TYPE_DIRECTFB,
	    CAIRO_SURFACE_TYPE_SVG, // SVG格式
	    CAIRO_SURFACE_TYPE_OS2,
	    CAIRO_SURFACE_TYPE_WIN32_PRINTING,
	    CAIRO_SURFACE_TYPE_QUARTZ_IMAGE,
	    CAIRO_SURFACE_TYPE_SCRIPT,
	    CAIRO_SURFACE_TYPE_QT,
	    CAIRO_SURFACE_TYPE_RECORDING,
	    CAIRO_SURFACE_TYPE_VG,
	    CAIRO_SURFACE_TYPE_GL,  // OpenGL后端，使用OpenGL接口(libGL和dir驱动接口)绘制，利用3D硬件加速，目前处于experimental状态
	    CAIRO_SURFACE_TYPE_DRM, // 绕过Mesa，直接使用DRM/DRI接口的后端，理论上效率比较OpenGL更高，但是接口和功能都不成熟，问题很多，而且厂商维护力度小，目前处于experimental状态
	    CAIRO_SURFACE_TYPE_TEE,
	    CAIRO_SURFACE_TYPE_XML,
	    CAIRO_SURFACE_TYPE_SKIA,
	    CAIRO_SURFACE_TYPE_SUBSURFACE,
	    CAIRO_SURFACE_TYPE_COGL
	} cairo_surface_type_t;

#如何使用Cairo？

Cairo为用户提供一套完整的API，通常通过动态库的方式提供，比如Linux环境中，库的名称通常叫libcairo，存放于/usr/lib64之类的目录中。

Cairo官方也提供了比较入门的tutorial：

[Cairo tutorial](http://cairographics.org/tutorial/)

有关Cairo的详细信息请参考官网：

[Cairo官网](http://cairographics.org/)

这里有一个简单的代码示例(GTK2+中的写法，GTK3+中会有差别，有兴趣可以尝试改造移植下)，供参考：

	#include <cairo.h>
	#include <gtk/gtk.h>
	
	static void do_drawing(cairo_t *);
	
	struct {
	  int count;
	  double coordx[100];
	  double coordy[100];
	} glob;
	
	static gboolean on_draw_event(GtkWidget *widget, cairo_t *cr, 
	    gpointer user_data)
	{
	  do_drawing(cr);
	
	  return FALSE;
	}
	
	static void do_drawing(cairo_t *cr)
	{
	  // 设置源，这里是后面画线时使用的颜色值。
	  cairo_set_source_rgb(cr, 0, 0, 0);
	  // 设置线的宽度
	  cairo_set_line_width(cr, 0.5);
	
	  int i, j;
	  for (i = 0; i <= glob.count - 1; i++ ) {
	      for (j = 0; j <= glob.count - 1; j++ ) {
			  // 将当前位置移至指定坐标，画线时需要先定位线的起点位置。
	          cairo_move_to(cr, glob.coordx[i], glob.coordy[i]);
			  // 画线(实际只是指定Path)，起点为上一句代码指定，这里指定终点
	          cairo_line_to(cr, glob.coordx[j], glob.coordy[j]);
	      }
	  }
	
	  glob.count = 0;
	  // 上面的cairo_move_to和cairo_line_to只是指定Path，这里将之前指定的Path绘制出来，不调用Stroke，线是画不出来的。
	  cairo_stroke(cr);    
	}
	
	static gboolean clicked(GtkWidget *widget, GdkEventButton *event,
	    gpointer user_data)
	{
	    if (event->button == 1) {
	        glob.coordx[glob.count] = event->x;
	        glob.coordy[glob.count++] = event->y;
	    }
	
	    if (event->button == 3) {
	        gtk_widget_queue_draw(widget);
	    }
	
	    return TRUE;
	}
	
	
	int main(int argc, char *argv[])
	{
	  GtkWidget *window;
	  GtkWidget *darea;
	  
	  glob.count = 0;
	
	  gtk_init(&argc, &argv);
	
	  window = gtk_window_new(GTK_WINDOW_TOPLEVEL);
	
	  darea = gtk_drawing_area_new();
	  gtk_container_add(GTK_CONTAINER(window), darea);
	 
	  gtk_widget_add_events(window, GDK_BUTTON_PRESS_MASK);
	  // 注册drawing_area控件的Draw事件处理函数。
	  g_signal_connect(G_OBJECT(darea), "draw", 
	      G_CALLBACK(on_draw_event), NULL); 
	  g_signal_connect(window, "destroy",
	      G_CALLBACK(gtk_main_quit), NULL);  
	    
	  g_signal_connect(window, "button-press-event", 
	      G_CALLBACK(clicked), NULL);
	 
	  gtk_window_set_position(GTK_WINDOW(window), GTK_WIN_POS_CENTER);
	  gtk_window_set_default_size(GTK_WINDOW(window), 400, 300); 
	  gtk_window_set_title(GTK_WINDOW(window), "Lines");
	
	  gtk_widget_show_all(window);
	
	  gtk_main();
	
	  return 0;
	}







