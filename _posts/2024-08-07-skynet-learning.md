---
layout:     post
title:      "skynet"
subtitle:   "skynet学习"
date:       2024-08-07 23:36:00
author:     "Yaoxh6"
catalog: true
header-style: text
tags:
  - skynet
  - game server
  - framework
---

# 启动流程
入口在skynet_main.c文件的main函数,
在必要的初始化之后调用`skynet_start`，这里为所有线程以及模块加载的入口。
## skynet_start
在这个函数里主要做了两件事情,第一初始化了logger模块，第二个是创建了多线程用于整个游戏的服务。
### logger
logger模块一般是最先启动的模块，通过`skynet_context_new`启动，其他的模块也是同样的启动方式。首先是`skynet_module_query`，这里的主要作用是通过名字找到对应的so文件，并且绑定对应的函数到module中。
```c
struct skynet_module {
	const char * name;
	void * module;
	skynet_dl_create create;
	skynet_dl_init init;
	skynet_dl_release release;
	skynet_dl_signal signal;
};
```
已`logger`为例，则会找到`logger.so`，并且将`service_logger.c`中的`logger_init`绑定为`module`中的`init`函数。`init`函数主要区分有无文件名字参数，并且绑定回调函数。启动`skynet`之后，所见到的日志即为`logger_cb`的调用。那`logger_cb`是如何调用的?`skynet`为actor模型，logger服务的调用来源是从消息队列取出并且执行。从头开始可追溯到`skynet_error`函数。首先通过`skynet_handle_findname`取得`logger`服务，在`skynet`中，`name`,`handle`以及具体的`server`是互相绑定的。经过对具体log的一些封装，最后调用`skynet_context_push`，放到消息队列中。`skynet`每个服务对应的消息队列是循环队列，通过有单向链表构成的全局队列。具体的`message`先入本地队列再将本地队列放入到全局队列中。至此，消息入队列。

消息从队列中到被执行，从头追溯可到`thread_worker`。`skynet_context_message_dispatch`，不断从全局队列中取出消息并且执行，首先`skynet_globalmq_pop`，然后通过`skynet_mq_handle`得到`handle`,得到`handle`就可以拿到具体的服务。不停的通过`skynet_mq_pop`将消息取出队列，并且判断`ctx->cb`是否存在并且执行,所以最后会执行到`logger_cb`。