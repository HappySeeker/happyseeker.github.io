---
layout: post
title:  "Maximum number of clients reached问题"
date:   2016-02-05 06:22:10
author: Jiang Biao
categories: Graphic
---

# 闲话

明天就要国内了，今天要早去公司，时间不多。 简单说说昨天写测试代码时遇到的一个问题吧。

# 问题现象

写了一个cairo+gtk2的测试代码，运行一段时间后终端中有报错：

	Maximum number of clients reached

咋一看，不是很明白其意思，这里的clients从第一印象来理解，应该是相对于X Server的客户端，应该是Xorg报错了。 

# 简单分析

搜了下Xorg的代码，果然：

	#define NOROOM "Maximum number of clients reached"

出错的流程为：

	EstablishNewConnections()->
	  ErrorConnMax()

即在建立新连接的时候，出现了错误：

	Bool
	EstablishNewConnections(ClientPtr clientUnused, void *closure)
	{
	...
	
	        if (!AllocNewConnection(new_trans_conn, newconn, connect_time)) {
	            ErrorConnMax(new_trans_conn);
	...
	}

再看看`AllocNewConnection`的实现：

	static ClientPtr
	AllocNewConnection(XtransConnInfo trans_conn, int fd, CARD32 conn_time)
	{
	    OsCommPtr oc;
	    ClientPtr client;
	
	    if (
	#ifndef WIN32
	           fd >= lastfdesc
	#else
	           XFD_SETCOUNT(&AllClients) >= MaxClients
	#endif
	        )
	        return NullClient;
	    oc = malloc(sizeof(OsCommRec));
	    if (!oc)
	        return NullClient;
	    oc->trans_conn = trans_conn;
	    oc->fd = fd;
	    oc->input = (ConnectionInputPtr) NULL;
	    oc->output = (ConnectionOutputPtr) NULL;
	    oc->auth_id = None;
	    oc->conn_time = conn_time;
	    if (!(client = NextAvailableClient((void *) oc))) {
	        free(oc);
	        return NullClient;
	    }
	    client->local = ComputeLocalClient(client);
	#if !defined(WIN32)
	    ConnectionTranslation[fd] = client->index;
	#else
	    SetConnectionTranslation(fd, client->index);
	#endif
	    if (GrabInProgress) {
	        FD_SET(fd, &SavedAllClients);
	        FD_SET(fd, &SavedAllSockets);
	    }
	    else {
	        FD_SET(fd, &AllClients);
	        FD_SET(fd, &AllSockets);
	    }
	
	#ifdef DEBUG
	    ErrorF("AllocNewConnection: client index = %d, socket fd = %d\n",
	           client->index, fd);
	#endif
	#ifdef XSERVER_DTRACE
	    XSERVER_CLIENT_CONNECT(client->index, fd);
	#endif
	
	    return client;
	}

从代码看，最可能返回NULL的就是连接数超过了MaxClients，MaxClients定义为256。

这里指的连接具体指什么？

我们知道X11环境中，采用的是CS模型，Xorg作为服务的，应用程序作为客户端，客户端(应用程序)发出绘图请求，然后由服务端(Xorg)负责处理请求，并执行最终的绘制操作，也就是说，应用程序如果需要做任何的绘图操作，都需要通过向Xorg发出请求来实现，客户端与服务端通信的首要条件就是要先创建连接，X11环境中的连接方式就是通过Socket而已，可以将其看作为通过socket通信的两端。

所以，这里的连接数操作限制，应该是指Xorg listen的Socket的连接数超过了256。

# 原因

通常什么原因会导致这样的情况呢？

最简单的案例就是open连接后，忘了close，出现连接泄露了。 那么客户端代码中哪些是负责创建和关闭连接的接口呢？

对于使用GLX接口来说，就是XOpenDisplay和XCloseDisplay了。

走查了下自己的代码(提供上层的库)，发现自己提供的接口中仅使用了XOpenDisplay打开连接，但没有直接使用XCloseDisplay关闭，但我注册了钩子，在cairo_t关闭的时候应该会自动调用XCloseDisplay关闭。

再走查上层应用的代码，才发现自己犯了个低级的错误，在cairo_t后，忘了使用cairo_destroy()销毁cr，最终导致问题。
