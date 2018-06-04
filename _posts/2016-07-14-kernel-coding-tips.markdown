---
layout: post
title:  "内核编码技巧"
date:   2016-07-14 06:13:09
author: HappySeeker
categories: Kernel
---

# 背景

内核中编码有很多跟用户态不一样的地方，需要有一些技巧，本文做相关记录，不全面，随性更新。。。

# 内核中新增源码目录的方法

最近移植新驱动到老版本内核的过程中，发现新增的目录没有编译，尝试解决了，记录下主要步骤：

1. 父目录的Makefile中添加obj-y=<子目录>
2. 父目录的Kconfig中添加source <子目录Kconfig>
3. 子目录中添加Kconfig文件，其中按规范格式(参考其他目录中的编写方法)编写相应的配置项。
4. 内核源码目录中执行menuconfig进行相应配置，操作后配置更新到.config文件中(当然也可以手工修改.config配置)

# rcu相关使用方法

rcu优雅周期的开始：
	rcu_read_lock()
优雅周期的结束：
	rcu_read_unlock()

在优雅周期内的所有读操作都是安全的，不会产生死锁。

## 链表相关的rcu操作

1、订阅(读链表。本质为引用：rcu_dereference_raw)，在非抢占内核中只是一些屏障，防止乱序。

	list_for_each_entry_rcu
	list_entry_rcu
	hlist_for_each_entry_rcu
	hlist_for_each_entry_rcu_bh
	...

2、发布(修改链表的相关操作)。关键是要等待优雅周期结束，在所有CPU上的优雅周期结束（`rcu_read_unlock()`）后，才能修改。所以，上述的订阅操作必须结合`rcu_read_lock()`和`rcu_read_unlock()`使用，否则就没有效果了。
	
	list_add_rcu
	list_add_tail_rcu
	list_del_rcu
	hlist_del_init_rcu
	list_replace_rcu

## 标准用法示例

订阅：

	rcu_read_lock();
	list_for_each_entry_rcu(hard_iface, &batadv_hardif_list, list) {
		if (hard_iface->soft_iface != soft_iface)
			continue;

		batadv_iv_ogm_send_to_if(forw_packet, hard_iface);
	}
	rcu_read_unlock();

相应的发布：

	list_add_tail_rcu(&hard_iface->list, &batadv_hardif_list);

# 链表遍历之**Safe**

list遍历通常使用接口：

	list_for_each_safe
	list_for_each

这里的**safe**容易误导使用者，很多人认为用了safe就真的“safe”了。

其实不然，safe的用处在于：当您需要在遍历链表的同时，需要删除(或修改)该链表中的节点时，需要用safe，比如：当删除当前节点时，会将当前的节点的pre和next指向非法地址或者是自己，这样会导致空指针panic或者死循环。

加了safe中，多增加了一个中间节点n，在遍历时会先取出next节点放在n中，当pos移向next节点时，就不会因为原来的节点被删除导致panic或死锁。

所以，当在遍历链表过程中需要删除(或修改)该链表中的节点时，才需要用safe，否则不需要。示例如下：

	rtnl_lock();
	list_for_each_entry_safe(hard_iface, hard_iface_tmp,
				 &batadv_hardif_list, list) {
		list_del_rcu(&hard_iface->list);
		batadv_hardif_remove_interface(hard_iface);
	}
	rtnl_unlock();

**注意**：使用safe只能防止自己对链表的修改，不能防止其它CPU上对该链表的并发修改。

所以，如果可能在多CPU上并发访问该链表时，应该使用其它的互斥保护机制，比如rcu就是专门针对指针保护可设计的，针对链表有单独的一套接口，是最安全，也是效率最高的方式，但其只能保护指针，其它内容不能保护。当然你也可能使用其它诸如spinlock的机制。

rcu遍历链表的接口如：

	list_for_each_entry_rcu

# Skb之引用计数

在多个上下文中同时使用skb时，要注意增加引用计数，否则在其它异步的场景中(比如软中断)可能会将skb释放掉，而你还在使用，结果可能导致：

1. skb重复释放
2. 内存乱踩

此时内核就肯定会出问题了。

skb_get接口用于增加skb的引用计数.

当然，用完了记得需要减掉引用计数，否则会导致skb无法释放，内存泄露。

- kfree_skb //减去引用计数，如果为0，则释放
- __kfree_skb  //不操作引用计数，直接释放，很危险，慎用。

当然，引用计数不限于对skb的保护，内核中其它很多公用的数据结构(如：task_struct\pid等)都需要引用计数保护。

# 内核补丁合入RPM包

在以RPM包方式维护的内核源码中，修改内核后，如何将相应的修改合入RPM包，以便于制作新的kernel RPM包？

步骤如下：

1、安装srpm包

  	rpm -ivh kernel-<ver>.src.rpm

2、进入rpmbuild目录(redhat系环境默认为/root/rpmbuild目录)，给内核打补丁
  
	rpmbuild -bp SPECS/kernel.spec

3、修改内核，并制作补丁

	cd BUILD/kernel-<ver>
	cp -fr linux-<ver> linux-<ver>.new
    cd linux-<ver>.new
    修改内核
    diff -auNr linux-<ver> linux-<ver>.new > ../SOURCES/xxx.patch

4、修改kernel.spec文件，加入新的补丁。新增代码示例如下：

	Patch900025: xxx.patch
    ...
	ApplyOptionalPatch xxx.patch

5、重新编译内核RPM包

    rpmbuild -ba SPECS/kernel.spec
    