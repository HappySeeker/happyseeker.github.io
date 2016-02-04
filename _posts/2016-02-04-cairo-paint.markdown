---
layout: post
title:  "cairo_paint()流程分析"
date:   2016-02-04 06:17:05
author: Jiang Biao
categories: Graphic
---

# 闲话

cairo是当前GTK环境中的绘图核心，其自身的绘图性能直接关乎整个图形环境的绘图性能，而Cairo提供了多种不同后端的支持，值得深入研究。

本文主要从cairo_paint()流程入手，分析使用OpenGL后端情况下的相关主要流程。 由于涉及的东西比较多，很多地方只能点到为止，后面有机会再深入探讨。

# 流程分析

## 流程概述

先从宏观上描述cairo_paint()的主要流程：

- cairo_paint() //入口
- _cairo_default_context_paint() // cairo_t结构中的backend的paint()接口
- _cairo_gstate_paint()
- _cairo_surface_paint() // 从gstate结构中取出surface，针对surface调用paint接口
- _cairo_gl_surface_paint() // cairo_surface_t结构中的backend的paint()接口
- _cairo_compositor_paint() // 合成器的paint()接口
- _cairo_spans_compositor_paint() // Spans合成器(看似默认的合成器)情况下对应的paint接口
- clip_and_composite_boxes() // 将要绘制的窗口转换成一个box的集合，然后绘制所有的box
- composite_boxes() //绘制boxes
- emit_aligned_boxes()
- _cairo_gl_composite_emit_rect()
- _cairo_gl_composite_flush() //触发flush操作，将缓冲区中的数据绘制到窗口上。这里很关键，后面再详述
- _cairo_gl_composite_draw_triangles_with_clip_region() or _cairo_gl_composite_draw_tristrip() //绘制一个个矩形区域或者tristrip区域
- _cairo_gl_composite_draw_triangles // 绘制矩形
- glDrawArrays() // 调用OpenGL接口，执行实际的绘制操作。

## \_cairo\_default\_context\_paint

从cairo_paint()入口进入后，调用了cairo_t结构中的backend的paint()接口，cairo_t中的backend为cairo_backend_t结构体，定义如下：
	
	struct _cairo {
	    cairo_reference_count_t ref_count;
	    cairo_status_t status;
	    cairo_user_data_array_t user_data;
	
	    const cairo_backend_t *backend;
	};

cairo_backend_t定义：

	typedef struct _cairo_backend cairo_backend_t;

包含所有的后端相关的操作：

	struct _cairo_backend {
	    cairo_backend_type_t type;
	    void (*destroy) (void *cr);
	
	    cairo_surface_t *(*get_original_target) (void *cr);
	    cairo_surface_t *(*get_current_target) (void *cr);
	
	    cairo_status_t (*save) (void *cr);
	    cairo_status_t (*restore) (void *cr);
	
	    cairo_status_t (*push_group) (void *cr, cairo_content_t content);
	    cairo_pattern_t *(*pop_group) (void *cr);
	
	    cairo_status_t (*set_source_rgba) (void *cr, double red, double green, double blue, double alpha);
	    cairo_status_t (*set_source_surface) (void *cr, cairo_surface_t *surface, double x, double y);
	    cairo_status_t (*set_source) (void *cr, cairo_pattern_t *source);
	    cairo_pattern_t *(*get_source) (void *cr);
	
	    cairo_status_t (*set_antialias) (void *cr, cairo_antialias_t antialias);
	    cairo_status_t (*set_dash) (void *cr, const double *dashes, int num_dashes, double offset);
	    cairo_status_t (*set_fill_rule) (void *cr, cairo_fill_rule_t fill_rule);
	    cairo_status_t (*set_line_cap) (void *cr, cairo_line_cap_t line_cap);
	    cairo_status_t (*set_line_join) (void *cr, cairo_line_join_t line_join);
	    cairo_status_t (*set_line_width) (void *cr, double line_width);
	    cairo_status_t (*set_miter_limit) (void *cr, double limit);
	    cairo_status_t (*set_opacity) (void *cr, double opacity);
	    cairo_status_t (*set_operator) (void *cr, cairo_operator_t op);
	    cairo_status_t (*set_tolerance) (void *cr, double tolerance);
	
	    cairo_antialias_t (*get_antialias) (void *cr);
	    void (*get_dash) (void *cr, double *dashes, int *num_dashes, double *offset);
	    cairo_fill_rule_t (*get_fill_rule) (void *cr);
	    cairo_line_cap_t (*get_line_cap) (void *cr);
	    cairo_line_join_t (*get_line_join) (void *cr);
	    double (*get_line_width) (void *cr);
	    double (*get_miter_limit) (void *cr);
	    double (*get_opacity) (void *cr);
	    cairo_operator_t (*get_operator) (void *cr);
	    double (*get_tolerance) (void *cr);
	
	    cairo_status_t (*translate) (void *cr, double tx, double ty);
	    cairo_status_t (*scale) (void *cr, double sx, double sy);
	    cairo_status_t (*rotate) (void *cr, double theta);
	    cairo_status_t (*transform) (void *cr, const cairo_matrix_t *matrix);
	    cairo_status_t (*set_matrix) (void *cr, const cairo_matrix_t *matrix);
	    cairo_status_t (*set_identity_matrix) (void *cr);
	    void (*get_matrix) (void *cr, cairo_matrix_t *matrix);
	
	    void (*user_to_device) (void *cr, double *x, double *y);
	    void (*user_to_device_distance) (void *cr, double *x, double *y);
	    void (*device_to_user) (void *cr, double *x, double *y);
	    void (*device_to_user_distance) (void *cr, double *x, double *y);
	
	    void (*user_to_backend) (void *cr, double *x, double *y);
	    void (*user_to_backend_distance) (void *cr, double *x, double *y);
	    void (*backend_to_user) (void *cr, double *x, double *y);
	    void (*backend_to_user_distance) (void *cr, double *x, double *y);
	
	    cairo_status_t (*new_path) (void *cr);
	    cairo_status_t (*new_sub_path) (void *cr);
	    cairo_status_t (*move_to) (void *cr, double x, double y);
	    cairo_status_t (*rel_move_to) (void *cr, double dx, double dy);
	    cairo_status_t (*line_to) (void *cr, double x, double y);
	    cairo_status_t (*rel_line_to) (void *cr, double dx, double dy);
	    cairo_status_t (*curve_to) (void *cr, double x1, double y1, double x2, double y2, double x3, double y3);
	    cairo_status_t (*rel_curve_to) (void *cr, double dx1, double dy1, double dx2, double dy2, double dx3, double dy3);
	    cairo_status_t (*arc_to) (void *cr, double x1, double y1, double x2, double y2, double radius);
	    cairo_status_t (*rel_arc_to) (void *cr, double dx1, double dy1, double dx2, double dy2, double radius);
	    cairo_status_t (*close_path) (void *cr);
	
	    cairo_status_t (*arc) (void *cr, double xc, double yc, double radius, double angle1, double angle2, cairo_bool_t forward);
	    cairo_status_t (*rectangle) (void *cr, double x, double y, double width, double height);
	
	    void (*path_extents) (void *cr, double *x1, double *y1, double *x2, double *y2);
	    cairo_bool_t (*has_current_point) (void *cr);
	    cairo_bool_t (*get_current_point) (void *cr, double *x, double *y);
	
	    cairo_path_t *(*copy_path) (void *cr);
	    cairo_path_t *(*copy_path_flat) (void *cr);
	    cairo_status_t (*append_path) (void *cr, const cairo_path_t *path);
	
	    cairo_status_t (*stroke_to_path) (void *cr);
	
	    cairo_status_t (*clip) (void *cr);
	    cairo_status_t (*clip_preserve) (void *cr);
	    cairo_status_t (*in_clip) (void *cr, double x, double y, cairo_bool_t *inside);
	    cairo_status_t (*clip_extents) (void *cr, double *x1, double *y1, double *x2, double *y2);
	    cairo_status_t (*reset_clip) (void *cr);
	    cairo_rectangle_list_t *(*clip_copy_rectangle_list) (void *cr);
	
	    cairo_status_t (*paint) (void *cr);
	    cairo_status_t (*paint_with_alpha) (void *cr, double opacity);
	    cairo_status_t (*mask) (void *cr, cairo_pattern_t *pattern);
	
	    cairo_status_t (*stroke) (void *cr);
	    cairo_status_t (*stroke_preserve) (void *cr);
	    cairo_status_t (*in_stroke) (void *cr, double x, double y, cairo_bool_t *inside);
	    cairo_status_t (*stroke_extents) (void *cr, double *x1, double *y1, double *x2, double *y2);
	
	    cairo_status_t (*fill) (void *cr);
	    cairo_status_t (*fill_preserve) (void *cr);
	    cairo_status_t (*in_fill) (void *cr, double x, double y, cairo_bool_t *inside);
	    cairo_status_t (*fill_extents) (void *cr, double *x1, double *y1, double *x2, double *y2);
	
	    cairo_status_t (*set_font_face) (void *cr, cairo_font_face_t *font_face);
	    cairo_font_face_t *(*get_font_face) (void *cr);

GL后端没有单独定义的backend，有默认的backend可用：

	static const cairo_backend_t _cairo_default_context_backend = {
	    CAIRO_TYPE_DEFAULT,
	    _cairo_default_context_destroy,
	
	    _cairo_default_context_get_original_target,
	    _cairo_default_context_get_current_target,
	
	    _cairo_default_context_save,
	    _cairo_default_context_restore,
	
	    _cairo_default_context_push_group,
	    _cairo_default_context_pop_group,
	
	    _cairo_default_context_set_source_rgba,
	    _cairo_default_context_set_source_surface,
	...
	    _cairo_default_context_paint,
	    _cairo_default_context_paint_with_alpha,
	    _cairo_default_context_mask,
	...
	}

可见，其paint()接口为_cairo_default_context_paint。

## _cairo_default_context_paint

	static cairo_status_t
	_cairo_default_context_paint (void *abstract_cr)
	{
	    cairo_default_context_t *cr = abstract_cr;
	
	    return _cairo_gstate_paint (cr->gstate);
	}

_cairo_default_context_paint()中，将cairo_t结构指针转换为cairo_default_context_t结构指针，cairo_default_context_t实际为cairo_t的子类，这是一种典型的在C中的"多态"的实现方式，即使用父类(抽象类)指针指向子类，从而使用相同的接口(抽象接口)，调用不同子类的接口。 具体来说，如果有不同的cairo_backend_t实现，则不同的backend的各接口中，则可以使用cairo_t的不同的子类(调用时通过传入不同子类作为实参即可)对应的接口，从而实现“多态。”

Cairo中类似的实现手法还有很多，代码看多了就会慢慢体会到。 这也给大家提了个醒，要用OO(面向对象)的思维方式来阅读代码，这样才能更容易理解代码，理解代码逻辑和本质。 虽然Cairo是基于纯C语言实现的，但是其设计都是基于OO的思想来设计的，如果以C的方式去理解这些代码，在一些关键的地方就会觉得很别扭、很迷惑，换个角度(OO)思考问题，就会豁然开朗，原来如此而已，只是由于要C中实现OO，会比较麻烦，所以，很多代码实现看起来都很别扭，但如果不关心这些实现细节，从更高的层次来看，就很容易理解了。 比如这里经常使用的各种abstract对象指针，各个结构体中的base成员，其实都只是为了实现C++的类的继承关系和多态而已。

指针转换后，直接调用了_cairo_gstate_paint接口。

## \_cairo\_gstate\_paint

	cairo_status_t
	_cairo_surface_paint (cairo_surface_t		*surface,
			      cairo_operator_t		 op,
			      const cairo_pattern_t	*source,
			      const cairo_clip_t	*clip)
	{
	...
	
	    return _cairo_surface_set_error (surface, status);
	}

这个函数没有特别的地方需要讲解，主要是做一些状态和变量检查，然后调用_cairo_surface_paint()接口，传入的参数从之前的cairo_t类型变为了cairo_surface_t，即从原来的cr变成了surface，surface和cr的关系如下：

	surface = cr->gstate->target

## \_cairo\_surface\_paint

	cairo_status_t
	_cairo_surface_paint (cairo_surface_t		*surface,
			      cairo_operator_t		 op,
			      const cairo_pattern_t	*source,
			      const cairo_clip_t	*clip)
	{
	...
	
	    status = surface->backend->paint (surface, op, source, clip);
	...
	}

这个函数也比较赶紧，前面做了相关的状态检查和简单的准备后，调用了surface中backend成员的paint接口。

注意：这里的backend跟cairo_t结构的backend是不同的，不要混淆了。这里的backend定义为：

	typedef struct _cairo_surface_backend cairo_surface_backend_t;

_cairo_surface_backend 定义：

	struct _cairo_surface_backend {
	    cairo_surface_type_t type;
	
	    cairo_warn cairo_status_t
	    (*finish)			(void			*surface);
	
	    cairo_t *
	    (*create_context)		(void			*surface);
	
	    cairo_surface_t *
	    (*create_similar)		(void			*surface,
					 cairo_content_t	 content,
					 int			 width,
					 int			 height);
	    cairo_surface_t *
	    (*create_similar_image)	(void			*surface,
					 cairo_format_t		format,
					 int			 width,
					 int			 height);
	
	    cairo_image_surface_t *
	    (*map_to_image)		(void			*surface,
					 const cairo_rectangle_int_t  *extents);
	    cairo_int_status_t
	    (*unmap_image)		(void			*surface,
					 cairo_image_surface_t	*image);
	
	    cairo_surface_t *
	    (*source)			(void                    *abstract_surface,
					 cairo_rectangle_int_t  *extents);
	
	    cairo_warn cairo_status_t
	    (*acquire_source_image)	(void                    *abstract_surface,
					 cairo_image_surface_t  **image_out,
					 void                   **image_extra);
	
	    cairo_warn void
	    (*release_source_image)	(void                   *abstract_surface,
					 cairo_image_surface_t  *image_out,
					 void                   *image_extra);
	
	    cairo_surface_t *
	    (*snapshot)			(void			*surface);
	
	    cairo_warn cairo_int_status_t
	    (*copy_page)		(void			*surface);
	
	    cairo_warn cairo_int_status_t
	    (*show_page)		(void			*surface);
	
	    /* Get the extents of the current surface. For many surface types
	     * this will be as simple as { x=0, y=0, width=surface->width,
	     * height=surface->height}.
	     *
	     * If this function is not implemented, or if it returns
	     * FALSE the surface is considered to be
	     * boundless and infinite bounds are used for it.
	     */
	    cairo_bool_t
	    (*get_extents)		(void			 *surface,
					 cairo_rectangle_int_t   *extents);
	
	    void
	    (*get_font_options)         (void                  *surface,
					 cairo_font_options_t  *options);
	
	    cairo_warn cairo_status_t
	    (*flush)                    (void                  *surface,
					 unsigned               flags);
	
	    cairo_warn cairo_status_t
	    (*mark_dirty_rectangle)     (void                  *surface,
					 int                    x,
					 int                    y,
					 int                    width,
					 int                    height);
	
	    cairo_warn cairo_int_status_t
	    (*paint)			(void			*surface,
					 cairo_operator_t	 op,
					 const cairo_pattern_t	*source,
					 const cairo_clip_t		*clip);
	
	  ...
	}

这个backend中实现的还是一堆的绘图操作。可以这样理解，surface中的backend(_cairo_surface_backend)是针对surface的一些接口，比如paint、create_context;而cairo_t中的backend更具体，实现的是具体的绘图操作，比如画线、画圆、画矩形。

这里之所以要抽象出backend接口，是因为Cairo需要支持不同的后端，不同的后端相关接口的具体实现不同，所以如此抽象，这是典型的OO思想中的依赖倒置原则，让上层依赖抽象接口，而不依赖具体的底层实现细节。

OpenGL的后端对应的backend为：cairo_surface_backend_t

定义如下：
	static const cairo_surface_backend_t _cairo_gl_surface_backend = {
	    CAIRO_SURFACE_TYPE_GL,
	    _cairo_gl_surface_finish,
	    _cairo_default_context_create,
	
	    _cairo_gl_surface_create_similar,
	    NULL, /* similar image */
	    _cairo_gl_surface_map_to_image,
	    _cairo_gl_surface_unmap_image,
	
	    _cairo_gl_surface_source,
	    _cairo_gl_surface_acquire_source_image,
	    _cairo_gl_surface_release_source_image,
	    NULL, /* snapshot */
	
	    NULL, /* copy_page */
	    NULL, /* show_page */
	
	    _cairo_gl_surface_get_extents,
	    _cairo_image_surface_get_font_options,
	
	    _cairo_gl_surface_flush,
	    NULL, /* mark_dirty_rectangle */
		...
	
	    _cairo_gl_surface_paint,
	    _cairo_gl_surface_mask,
	    _cairo_gl_surface_stroke,
	    _cairo_gl_surface_fill,
	    NULL, /* fill/stroke */
	    _cairo_gl_surface_glyphs,
	};

所以，paint对应的接口为：_cairo_gl_surface_paint

## \_cairo\_gl\_surface\_paint

	/*OpenGL后端的paint接口的入口*/
	static cairo_int_status_t
	_cairo_gl_surface_paint (void			*surface,
				 cairo_operator_t	 op,
				 const cairo_pattern_t	*source,
				 const cairo_clip_t	*clip)
	{
	    /* simplify the common case of clearing the surface */
		/*检查是否是最简单clear操作，如果是，则调用clear相关接口*/
	    if (clip == NULL) {
	        if (op == CAIRO_OPERATOR_CLEAR)
	            return _cairo_gl_surface_clear (surface, CAIRO_COLOR_TRANSPARENT);
	       else if (source->type == CAIRO_PATTERN_TYPE_SOLID &&
	                (op == CAIRO_OPERATOR_SOURCE ||
	                 (op == CAIRO_OPERATOR_OVER && _cairo_pattern_is_opaque_solid (source)))) {
	            return _cairo_gl_surface_clear (surface,
	                                            &((cairo_solid_pattern_t *) source)->color);
	        }
	    }
		//调用合成接口
	    return _cairo_compositor_paint (get_compositor (surface), surface,
					    op, source, clip);
	}

_cairo_gl_surface_paint实现很简单，先检查是否是clear操作，如果是则调用_cairo_gl_surface_clear，如果不是，则调用合成器的paint接口：_cairo_compositor_paint

## \_cairo\_compositor\_paint

	cairo_int_status_t
	_cairo_compositor_paint (const cairo_compositor_t	*compositor,
				 cairo_surface_t		*surface,
				 cairo_operator_t		 op,
				 const cairo_pattern_t		*source,
				 const cairo_clip_t		*clip)
	{
	    cairo_composite_rectangles_t extents;
	    cairo_int_status_t status;
	
	    TRACE ((stderr, "%s\n", __FUNCTION__));
		/*初始化欲绘制的矩形区域，其实就是初始化extents，extents中包含了suface、operation和绘制的区域等所有信息*/    
	    status = _cairo_composite_rectangles_init_for_paint (&extents, surface,
								 op, source,
								 clip);
	    if (unlikely (status))
		return status;
	
	    do {
		while (compositor->paint == NULL)
		    compositor = compositor->delegate;
		/*调用合成器对应的paint接口*/
		status = compositor->paint (compositor, &extents);
	
		compositor = compositor->delegate;
	    } while (status == CAIRO_INT_STATUS_UNSUPPORTED);
	
	    if (status == CAIRO_INT_STATUS_SUCCESS && surface->damage) {
		TRACE ((stderr, "%s: applying damage (%d,%d)x(%d, %d)\n",
			__FUNCTION__,
			extents.unbounded.x, extents.unbounded.y,
			extents.unbounded.width, extents.unbounded.height));
		surface->damage = _cairo_damage_add_rectangle (surface->damage,
							       &extents.unbounded);
	    }
	
	    _cairo_composite_rectangles_fini (&extents);
	
	    return status;
	}

这个函数主要做了两件事：

1. 初始化欲绘制的矩形区域，其实就是初始化extents，extents中包含了suface、operation和绘制的区域等所有信息。 extents是cairo_composite_rectangles_t结构体类型，具体定义就不详解了，内容太多，我们只要了解，绘图需要的所有信息都在这个里面就可以了。
2. 调用compositor的paint接口，又一次抽象了接口，抽象了对象(合成器)，典型的依赖倒置。 在C语言中，本质就是函数钩子。

### 关于合成器(compositor)

cairo中的compositor定义如下：

	typedef struct cairo_compositor cairo_compositor_t;

对应结构体为：

	struct cairo_compositor {
	    const cairo_compositor_t *delegate;
	
	    cairo_warn cairo_int_status_t
	    (*paint)			(const cairo_compositor_t	*compositor,
					 cairo_composite_rectangles_t	*extents);
	
	    cairo_warn cairo_int_status_t
	    (*mask)			(const cairo_compositor_t	*compositor,
					 cairo_composite_rectangles_t	*extents);
	...
	}

同样，合成器对应的结构体中仍然只是一个函数钩子，从OO角度看，就是一些抽象接口，不同的合成器(cairo_compositor的子类)可以有不同的实现，但对上层来说接口是统一的，松耦合。

### 合成器的初始化

合成器的初始化主要在如下函数中：

	/*OpenGL后端的cairo context初始化*/
	cairo_status_t
	_cairo_gl_context_init (cairo_gl_context_t *ctx)
	{
		...
	
	    _cairo_device_init (&ctx->base, &_cairo_gl_device_backend);
	
	    /* XXX The choice of compositor should be made automatically at runtime.
	     * However, it is useful to force one particular compositor whilst
	     * testing.
	     *几种合成器有什么区别?*/
	     if (_cairo_gl_msaa_compositor_enabled ())
		ctx->compositor = _cairo_gl_msaa_compositor_get ();
	    else
		ctx->compositor = _cairo_gl_span_compositor_get ();
		...
	}

可见默认选的是span合成器，得到的是cairo_spans_compositor_t类型的结构体。
再看看后续的初始化流程：

	_cairo_gl_span_compositor_get ->
	  _cairo_spans_compositor_init ->

	void
	_cairo_spans_compositor_init (cairo_spans_compositor_t *compositor,
				      const cairo_compositor_t  *delegate)
	{
	    compositor->base.delegate = delegate;
	
	    compositor->base.paint  = _cairo_spans_compositor_paint;
	    compositor->base.mask   = _cairo_spans_compositor_mask;
	    compositor->base.fill   = _cairo_spans_compositor_fill;
	    compositor->base.stroke = _cairo_spans_compositor_stroke;
	    compositor->base.glyphs = NULL;
	}

可见其paint接口定义为_cairo_spans_compositor_paint。

## \_cairo\_spans\_compositor\_paint()
	
	/* high-level compositor interface */
	
	static cairo_int_status_t
	_cairo_spans_compositor_paint (const cairo_compositor_t		*_compositor,
				       cairo_composite_rectangles_t	*extents)
	{
	    const cairo_spans_compositor_t *compositor = (cairo_spans_compositor_t*)_compositor;
	    cairo_boxes_t boxes;
	    cairo_int_status_t status;
	
	    TRACE ((stderr, "%s\n", __FUNCTION__));
	    _cairo_clip_steal_boxes (extents->clip, &boxes);
	    status = clip_and_composite_boxes (compositor, extents, &boxes);
	    _cairo_clip_unsteal_boxes (extents->clip, &boxes);
	
	    return status;
	}

这个函数实现也很简单，是合成器的高层接口，主要就是调用了clip_and_composite_boxes接口。

## clip\_and\_composite\_boxes

	static cairo_int_status_t
	clip_and_composite_boxes (const cairo_spans_compositor_t	*compositor,
				  cairo_composite_rectangles_t		*extents,
				  cairo_boxes_t				*boxes)
	{
		...
		/*将extents拆分为boxes?*/
	    status = trim_extents_to_boxes (extents, boxes);
	    if (unlikely (status))
			return status;
	
		...
	    status = composite_boxes (compositor, extents, boxes);
	    if (status != CAIRO_INT_STATUS_UNSUPPORTED)
			return status;
		...
	}

该函数主要做了：

1. 将extents转换成boxes。box就是最终绘图的基本单元了。
2. 调用composite_boxes，绘制box。

## composite\_boxes

主要调用了emit_aligned_boxes，其他不详述了。

## emit\_aligned\_boxes

	emit_aligned_boxes (cairo_gl_context_t *ctx,
			    const cairo_boxes_t *boxes)
	{
	    const struct _cairo_boxes_chunk *chunk;
	    cairo_gl_emit_rect_t emit = _cairo_gl_context_choose_emit_rect (ctx);
	    int i;
	
	    TRACE ((stderr, "%s: num_boxes=%d\n", __FUNCTION__, boxes->num_boxes));
	    for (chunk = &boxes->chunks; chunk; chunk = chunk->next) {
		for (i = 0; i < chunk->count; i++) {
		    int x1 = _cairo_fixed_integer_part (chunk->base[i].p1.x);
		    int y1 = _cairo_fixed_integer_part (chunk->base[i].p1.y);
		    int x2 = _cairo_fixed_integer_part (chunk->base[i].p2.x);
		    int y2 = _cairo_fixed_integer_part (chunk->base[i].p2.y);
		    emit (ctx, x1, y1, x2, y2);
		}
	    }
	}

通过`_cairo_gl_context_choose_emit_rect`获取emit的操作，这里默认为：`_cairo_gl_composite_emit_rect`，然后就调用它了。

## \_cairo\_gl\_composite\_emit\_rect

	static void
	_cairo_gl_composite_emit_rect (cairo_gl_context_t *ctx,
	                               GLfloat x1, GLfloat y1,
	                               GLfloat x2, GLfloat y2)
	{
		/*准备vbo buffer，如果发现buffer中空间不足，则先flush*/
	    _cairo_gl_composite_prepare_buffer (ctx, 6,
						CAIRO_GL_PRIMITIVE_TYPE_TRIANGLES);
		/*向ctx的vbo buffer中提交顶点数据，在下次flush时绘制*/
	    _cairo_gl_composite_emit_vertex (ctx, x1, y1);
	    _cairo_gl_composite_emit_vertex (ctx, x2, y1);
	    _cairo_gl_composite_emit_vertex (ctx, x1, y2);
	
	    _cairo_gl_composite_emit_vertex (ctx, x2, y1);
	    _cairo_gl_composite_emit_vertex (ctx, x2, y2);
	    _cairo_gl_composite_emit_vertex (ctx, x1, y2);
	}
		  
主要做两件事：
1. 准备buffer，如果发现buffer中空间不足，则先flush。
2. 向ctx的vbo buffer中提交顶点数据，在下次flush时绘制

## \_cairo\_gl\_composite\_prepare\_buffer

	static void
	_cairo_gl_composite_prepare_buffer (cairo_gl_context_t *ctx,
					    unsigned int n_vertices,
					    cairo_gl_primitive_type_t primitive_type)
	{
	    if (ctx->primitive_type != primitive_type) {
		_cairo_gl_composite_flush (ctx);
		ctx->primitive_type = primitive_type;
	    }
		/*判断vbo buffer中的空间是否足以放下新的数据，如果不足，则flush，先将原来buffer中的数据绘制*/
	    if (ctx->vb_offset + n_vertices * ctx->vertex_size > CAIRO_GL_VBO_SIZE)
		_cairo_gl_composite_flush (ctx);
	}

主要工作：判断vbo buffer中的空间是否足以放下新的数据，如果不足，则flush，先将原来buffer中的数据绘制。

## \_cairo\_gl\_composite\_flush

	void
	_cairo_gl_composite_flush (cairo_gl_context_t *ctx)
	{
	    unsigned int count;
	    int i;
	
	    if (_cairo_gl_context_is_flushed (ctx))
	        return;
	
	    count = ctx->vb_offset / ctx->vertex_size;
	    _cairo_gl_composite_unmap_vertex_buffer (ctx);
	
	    if (ctx->primitive_type == CAIRO_GL_PRIMITIVE_TYPE_TRISTRIPS) {
		_cairo_gl_composite_draw_tristrip (ctx);
	    } else {
		assert (ctx->primitive_type == CAIRO_GL_PRIMITIVE_TYPE_TRIANGLES);
		_cairo_gl_composite_draw_triangles_with_clip_region (ctx, count);
	    }
	
	    for (i = 0; i < ARRAY_LENGTH (&ctx->glyph_cache); i++)
		_cairo_gl_glyph_cache_unlock (&ctx->glyph_cache[i]);
	}

根据欲绘制元素的类型，调用`_cairo_gl_composite_draw_tristrip`或`_cairo_gl_composite_draw_triangles_with_clip_region`，后续以绘制矩形为例解释。

## \_cairo\_gl\_composite\_draw\_triangles\_with\_clip\_region

就不多说了，主要调用`_cairo_gl_composite_draw_triangles`

## \_cairo\_gl\_composite\_draw\_triangles

	static inline void
	_cairo_gl_composite_draw_triangles (cairo_gl_context_t *ctx,
					    unsigned int count)
	{
	    if (! ctx->pre_shader) {
	        glDrawArrays (GL_TRIANGLES, 0, count);
	    } else {
	        cairo_gl_shader_t *prev_shader = ctx->current_shader;
	
	        _cairo_gl_set_shader (ctx, ctx->pre_shader);
	        /*设置操作模式，主要是合成的模式*/
        	_cairo_gl_set_operator (ctx, CAIRO_OPERATOR_DEST_OUT, TRUE);
			/*调用OpenGL接口绘制图形，这里使用的是顶点队列的批量绘制接口，有助于提升性能*/
        	glDrawArrays (GL_TRIANGLES, 0, count);
	
	        _cairo_gl_set_shader (ctx, prev_shader);
	        _cairo_gl_set_operator (ctx, CAIRO_OPERATOR_ADD, TRUE);
	        glDrawArrays (GL_TRIANGLES, 0, count);
	    }
	}

这里实现最终的绘制操作，主要涉及两个动作：

1. 设置操作模式，主要是合成器的合成模式，合成器提供了十多种不同的模式，不同模式下合成效果不同，这里不关注具体的模式间的差别。
2. 调用OpenGL接口绘制图形，这里使用的是顶点队列的批量绘制接口，有助于提升性能。

# 总结

终于结束了，连我自己都感觉累了，流程确实有点长，而且中间隔离了很多层，很多钩子，代码不容易看，需要慢慢理解，还是要提醒，请用OO的眼光来看。

