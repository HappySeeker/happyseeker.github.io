---
layout: post
title:  "DRM后端Cairo-perf-trace工具崩溃Again"
date:   2016-01-27 14:06:59
author: Jiang Biao
categories: Graphic
---

# 问题

欲测试cairo DRM后端的性能，上次解决完radeon显卡drm backend中`create_context`函数钩子错位问题后，重新编译运行，发现继续崩溃。

这次直接core，没有提示信息，查看搜集到的core文件，现象如下：

	Program terminated with signal SIGSEGV, Segmentation fault.
	#0  0x000000fff19dc3f0 in _cairo_drm_surface_get_extents () from /lib64/libcairo.so.2
	(gdb) bt
	#0  0x000000fff19dc3f0 in _cairo_drm_surface_get_extents () from /lib64/libcairo.so.2
	#1  0x000000fff19a64f0 in _cairo_surface_paint () from /lib64/libcairo.so.2
	#2  0x000000fff19846a0 in _cairo_gstate_paint () from /lib64/libcairo.so.2
	#3  0x000000fff19420b0 in cairo_paint () from /lib64/libcairo.so.2
	#4  0x0000000120007264 in fill_surface (surface=<optimized out>)
	    at cairo-perf-trace.c:180
	#5  cairo_perf_trace.lto_priv.89 (perf=0xfffff94030, 
	    target=0x120024670 <targets.lto_priv>, 
	    trace=0xfffff973ee "../trimmed-cairo-traces-master/benchmark/t-gnome-terminal-vim")
	    at cairo-perf-trace.c:719
	#6  0x0000000120004df4 in main (argc=<optimized out>, argv=<optimized out>)
	    at cairo-perf-trace.c:1032

还是在`fill_surface()`函数中出了问题，这次是段错误。

# 分析

## 现场分析
还是先从故障现场开始分析：

	(gdb) disassemble 
	Dump of assembler code for function _cairo_drm_surface_get_extents:
	=> 0x000000fff19dc3f0 <+0>:	sw	zero,0(a1)
	   0x000000fff19dc3f4 <+4>:	li	v0,1
	   0x000000fff19dc3f8 <+8>:	sw	zero,4(a1)
	   0x000000fff19dc3fc <+12>:	lw	v1,332(a0)
	   0x000000fff19dc400 <+16>:	sw	v1,8(a1)
	   0x000000fff19dc404 <+20>:	lw	v1,336(a0)
	   0x000000fff19dc408 <+24>:	jr	ra
	   0x000000fff19dc40c <+28>:	sw	v1,12(a1)
	End of assembler dump.

可以看出是a1寄存器的值不对导致段错误，查看寄存器信息：

	(gdb) info registers 
	                  zero               at               v0               v1
	 R0   0000000000000000 0000000000000001 000000fff1a77e88 000000fff1a50e60 
	                    a0               a1               a2               a3
	 R4   000000012047af80 0000000000000002 000000fffff93930 0000000000000000 
	                    a4               a5               a6               a7
	 R8   0000000000000000 0000000000000000 3ff0000000000000 3fe0000000000000 
	                    t0               t1               t2               t3
	 R12  3ff0000000000000 0000000000000000 0000000000000000 8000400040004000 
	                    s0               s1               s2               s3
	 R16  000000012047af80 0000000000000000 000000fffff93930 0000000000000002 
	                    s4               s5               s6               s7
	 R20  0000000000000000 0000000000000000 000000fffff94030 0000000120020000 
	                    t8               t9               k0               k1
	 R24  0000000000000020 000000fff19dc3f0 0000000000000000 0000000000000000 
	                    gp               sp               s8               ra
	 R28  000000fff1a81840 000000fffff938f0 0000000120024670 000000fff19a64f0 
	---Type <return> to continue, or q <return> to quit---q

可以看出a1寄存器值为0000000000000002，显然作为指针访问的话，肯定会段错误。那这个值是如何来的呢？还是得分析代码流程。

## 代码分析

分析调用流程，发现`_cairo_surface_paint()`函数还是不太可能调用到`_cairo_drm_surface_get_extents()`函数中去，两者的功能都不同。

于是还是怀疑是跟之前类似的问题，再次确认`radeon_surface_backend`结构体的内容：

	static const cairo_surface_backend_t radeon_surface_backend = {
	    CAIRO_SURFACE_TYPE_DRM,
	    radeon_surface_finish,
	    _cairo_default_context_create,
	
	    radeon_surface_create_similar,
	
	    NULL,
	    radeon_surface_acquire_source_image,
	    radeon_surface_release_source_image,
	
	    NULL, NULL, NULL,
	    NULL, /* composite */
	    NULL, /* fill */
	    NULL, /* trapezoids */
	    NULL, /* span */
	    NULL, /* check-span */
	
	    NULL, /* copy_page */
	    NULL, /* show_page */
	    _cairo_drm_surface_get_extents,
	    NULL, /* old-glyphs */
	    _cairo_drm_surface_get_font_options,
	
	    radeon_surface_flush,
	    NULL, /* mark dirty */
	    NULL, NULL, /* font/glyph fini */
	
	    radeon_surface_paint,
	    radeon_surface_mask,
	    radeon_surface_stroke,
	    radeon_surface_fill,
	    radeon_surface_glyphs,
	};

数一下，`_cairo_drm_surface_get_extents`接口在该结构第18个字段，

再看看`cairo_surface_backend_t`的定义：

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
	
	    cairo_warn cairo_int_status_t
	    (*mask)			(void			*surface,
					 cairo_operator_t	 op,
					 const cairo_pattern_t	*source,
					 const cairo_pattern_t	*mask,
					 const cairo_clip_t		*clip);
	
	    cairo_warn cairo_int_status_t
	    (*stroke)			(void			*surface,
					 cairo_operator_t	 op,
					 const cairo_pattern_t	*source,
					 const cairo_path_fixed_t	*path,
					 const cairo_stroke_style_t	*style,
					 const cairo_matrix_t	*ctm,
					 const cairo_matrix_t	*ctm_inverse,
					 double			 tolerance,
					 cairo_antialias_t	 antialias,
					 const cairo_clip_t		*clip);
	
	    cairo_warn cairo_int_status_t
	    (*fill)			(void			*surface,
					 cairo_operator_t	 op,
					 const cairo_pattern_t	*source,
					 const cairo_path_fixed_t	*path,
					 cairo_fill_rule_t	 fill_rule,
					 double			 tolerance,
					 cairo_antialias_t	 antialias,
					 const cairo_clip_t           *clip);
	
	    cairo_warn cairo_int_status_t
	    (*fill_stroke)		(void			*surface,
					 cairo_operator_t	 fill_op,
					 const cairo_pattern_t	*fill_source,
					 cairo_fill_rule_t	 fill_rule,
					 double			 fill_tolerance,
					 cairo_antialias_t	 fill_antialias,
					 const cairo_path_fixed_t*path,
					 cairo_operator_t	 stroke_op,
					 const cairo_pattern_t	*stroke_source,
					 const cairo_stroke_style_t	*stroke_style,
					 const cairo_matrix_t	*stroke_ctm,
					 const cairo_matrix_t	*stroke_ctm_inverse,
					 double			 stroke_tolerance,
					 cairo_antialias_t	 stroke_antialias,
					 const cairo_clip_t	*clip);
	
	    cairo_warn cairo_int_status_t
	    (*show_glyphs)		(void			*surface,
					 cairo_operator_t	 op,
					 const cairo_pattern_t	*source,
					 cairo_glyph_t		*glyphs,
					 int			 num_glyphs,
					 cairo_scaled_font_t	*scaled_font,
					 const cairo_clip_t     *clip);
	
	    cairo_bool_t
	    (*has_show_text_glyphs)	(void			    *surface);
	
	    cairo_warn cairo_int_status_t
	    (*show_text_glyphs)		(void			    *surface,
					 cairo_operator_t	     op,
					 const cairo_pattern_t	    *source,
					 const char		    *utf8,
					 int			     utf8_len,
					 cairo_glyph_t		    *glyphs,
					 int			     num_glyphs,
					 const cairo_text_cluster_t *clusters,
					 int			     num_clusters,
					 cairo_text_cluster_flags_t  cluster_flags,
					 cairo_scaled_font_t	    *scaled_font,
					 const cairo_clip_t               *clip);
	
	    const char **
	    (*get_supported_mime_types)	(void			    *surface);
	};

数一下，该结构体的第18个字段的钩子为`paint`接口，这才是应该调用的接口，也就是说这个两个结构体又错位了，之前值修改了`create_context`接口，没有修改彻底，这次需要彻底核对一下了，核对后发现其中很多的钩子都没对上位，全部重新调整后解决。