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
在这个函数里主要做了两件事情,第一初始化了logger模块，第二个启动`bootstrap`,第三是创建了多线程用于整个游戏的服务。
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

### bootstrap
`bootstrap`默认参数使用config中的`snlua bootstrap`,同样使用`skynet_context_new`新建一个server, server为`snlua`，参数为`bootstrap`，直接跳转到`snlua_init`。首先`skynet_callback(ctx, l , launch_cb);`，设置自己的回调函数为`launch_cb`，然后调用`const char * self = skynet_command(ctx, "REG", NULL);`通过`skynet_command`调用到`cmd_reg`，并且拿回现在的`handle_id`，再加一为自己的`handle_id`，最后`skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);`给自己发送消息,`skynet_send`内部也会调用`skynet_context_push`,走的就是消息发送和接受那套流程。所以会调到`launch_cb`，这里清空回调函数，并且调用`init_cb`,传的参数是`bootstrap`.`init_cb`里面设置一些变量，最后调用`loader.lua`,参数是`bootstrap`,启动`bootstrap.lua`作为第一个服务。优先启动`skynet.launch("snlua","launcher")`launcher服务，并命名为`.launcher`,随后通过`skynet.newservice`启动`cmaster`，`cslave`，`datacenterd`，`service_mgr`，`main`。在`main.lua`中，再启动`protoloader`，`console`，`debug_console`，`simpledb`，`watchdog`。

有关`bootstrap`中的`start`函数调用流程。相当于调用`skynet.lua`中的函数.
```lua
function skynet.start(start_func)
	c.callback(skynet.dispatch_message)
	init_thread = skynet.timeout(0, function()
		skynet.init_service(start_func)
		init_thread = nil
	end)
end
```
首先将回调函数设置为`dispatch_message`,然后通过`skynet.timeout`触发，`local session = cintcommand("TIMEOUT", timeout)`发送TIMEOUT信号到队列，同时创建协程（有就复用）`co_create_for_timeout->co_create`，保存session和co,`session_id_coroutine[session] = co`。回调函数触发`raw_dispatch_message`，取出co,恢复执行。`local co = session_id_coroutine[session]`，`suspend(co, coroutine_resume(co, true, msg, sz, session))`.


`co_create`有两种情况, 从协程池中取出或者新建，一开始一定是新建，通过回调触发后会让出执行权`f = coroutine_yield "SUSPEND"`，并且保存在协程池中，下一次如果可以从协程池中取出，执行`coroutine_resume(co, f)`，那么`f = coroutine_yield "SUSPEND"`中的`f`就为想要resume的`f`，此时f还不会立刻执行，因为要等到消息回调触发，所以`f(coroutine_yield())`再次让出。
```lua
local function co_create(f)
	local co = tremove(coroutine_pool)
	if co == nil then
		co = coroutine_create(function(...)
			f(...)
			while true do
				local session = session_coroutine_id[co]
				if session and session ~= 0 then
					local source = debug.getinfo(f,"S")
					skynet.error(string.format("Maybe forgot response session %s from %s : %s:%d",
						session,
						skynet.address(session_coroutine_address[co]),
						source.source, source.linedefined))
				end
				-- coroutine exit
				local tag = session_coroutine_tracetag[co]
				if tag ~= nil then
					if tag then c.trace(tag, "end")	end
					session_coroutine_tracetag[co] = nil
				end
				local address = session_coroutine_address[co]
				if address then
					session_coroutine_id[co] = nil
					session_coroutine_address[co] = nil
				end

				-- recycle co into pool
				f = nil
				coroutine_pool[#coroutine_pool+1] = co
				-- recv new main function f
				f = coroutine_yield "SUSPEND"
				f(coroutine_yield())
			end
		end)
	else
		-- pass the main function f to coroutine, and restore running thread
		local running = running_thread
		coroutine_resume(co, f)
		running_thread = running
	end
	return co
end
```


# skynet.send,skynet.call和skynet.ret
## skynet.send
`skynet.send`相对简单，调用的是`c.send`, c又是`skynet.core`
``` lua
local c = require "skynet.core"
...
function skynet.send(addr, typename, ...)
	local p = proto[typename]
	return c.send(addr, p.id, 0 , p.pack(...))
end
```
所以直接找到`luaopen_skynet_core`,连同`send`还有其他一些函数。`send->lsend->send_message->skynet_send | skynet_sendname->skynet_send`,最后的入口就是`skynet_send`.

## skynet.call和skynet.ret
`skynet.call`相对复杂，顺带一提`skynet.newservice`是对`skynet.call`的简单封装。
``` lua
function skynet.newservice(name, ...)
	return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)
end

function skynet.call(addr, typename, ...)
	local tag = session_coroutine_tracetag[running_thread]
	if tag then
		c.trace(tag, "call", 2)
		c.send(addr, skynet.PTYPE_TRACE, 0, tag)
	end

	local p = proto[typename]
	local session = auxsend(addr, p.id , p.pack(...))
	if session == nil then
		error("call to invalid address " .. skynet.address(addr))
	end
	return p.unpack(yield_call(addr, session))
end
```
`skynet.call`主要包含两步，一步是`c.send`，一步是`p.unpack(yield_call(addr, session))`即返回回包。`yield_call`所得的结果来自`local succ, msg, sz = coroutine_yield "SUSPEND"`，之前说过回包是通过协程。回包使用的是`skynet.ret`，也是对`c.send`的封装，`local ret = c.send(co_address, skynet.PTYPE_RESPONSE, co_session, msg, sz)`，消息类型是`PTYPE_RESPONSE`，所以消息还是来自于`raw_dispatch_message`中的`suspend(co, coroutine_resume(co, true, msg, sz, session))`这里的`msg,sz`即为返回的值。所以不论是`callback`还是`call`都是通过协程返回。
```lua
local function raw_dispatch_message(prototype, msg, sz, session, source)
	-- skynet.PTYPE_RESPONSE = 1, read skynet.h
	if prototype == 1 then
		local co = session_id_coroutine[session]
		if co == "BREAK" then
			session_id_coroutine[session] = nil
		elseif co == nil then
			unknown_response(session, source, msg, sz)
		else
			local tag = session_coroutine_tracetag[co]
			if tag then c.trace(tag, "resume") end
			session_id_coroutine[session] = nil
			suspend(co, coroutine_resume(co, true, msg, sz, session))
		end
	else
		local p = proto[prototype]
		if p == nil then
			if prototype == skynet.PTYPE_TRACE then
				-- trace next request
				trace_source[source] = c.tostring(msg,sz)
			elseif session ~= 0 then
				c.send(source, skynet.PTYPE_ERROR, session, "")
			else
				unknown_request(session, source, msg, sz, prototype)
			end
			return
		end

		local f = p.dispatch
		if f then
			local co = co_create(f)
			session_coroutine_id[co] = session
			session_coroutine_address[co] = source
			local traceflag = p.trace
			if traceflag == false then
				-- force off
				trace_source[source] = nil
				session_coroutine_tracetag[co] = false
			else
				local tag = trace_source[source]
				if tag then
					trace_source[source] = nil
					c.trace(tag, "request")
					session_coroutine_tracetag[co] = tag
				elseif traceflag then
					-- set running_thread for trace
					running_thread = co
					skynet.trace()
				end
			end
			suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))
		else
			trace_source[source] = nil
			if session ~= 0 then
				c.send(source, skynet.PTYPE_ERROR, session, "")
			else
				unknown_request(session, source, msg, sz, proto[prototype].name)
			end
		end
	end
end

```

# skynet network
一开始skynet没有包含这部分，socket相关是后续补充而来。
network相关的操作大部分`require "skynet.socketdriver"`所以入口在 `luaopen_skynet_socketdriver`.

## 网络连接流程

### socket.listen

`llisten->skynet_socket_listen->socket_server_listen->send_request->write`,按照如此流程写入到管道.`request_package`包含`header`和`data`.
关于`socket`消息的读取，最早可追溯到`thread_socket`，`skynet_socket_poll`是无限循环，用来处理有关网络的数据，分为两步`has_cmd(ss)`第一用`select`单独处理`recvctrl_fd`（也用`sp_add`加入到了`epoll`）,第二用`ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);`封装了`epoll_wait`.对于`listen`事件，最终走`ctrl_cmd->listen_socket`.`skynet_socket_poll`处理完数据后，再通过`forward_message->skynet_context_push`给队列发送消息.

### socket.start和socket.connect
``` lua
function socket.start(id, func)
	driver.start(id)
	return connect(id, func)
end
```
`driverl.start`的流程

`driver.start->lstart->skynet_socket_start->socket_server_start->send_request(ss, &request, 'R', sizeof(request.u.resumepause));->resume_socket->forward_message(SKYNET_SOCKET_TYPE_CONNECT, true, &result);`

`connect`的流程`

`connect->suspend->skynet.wait->suspend_sleep->coroutine_yield "SUSPEND"`
所以connect的流程会在用协程让出控制权，直到`start`流程中的消息被处理，唤醒协程.

这里的协程不同于之前的`PTYPE_RESPONSE`,走`PTYPE_SOCKET`对应的分支`local f = p.dispatch`和`suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))`.
``` lua
skynet.register_protocol {
	name = "socket",
	id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
	unpack = driver.unpack,
	dispatch = function (_, _, t, ...)
		socket_message[t](...)
	end
}
```
先创建了`dispatch`的协程，根据协议`SKYNET_SOCKET_TYPE_CONNECT`,执行下面函数，
``` lua
-- SKYNET_SOCKET_TYPE_CONNECT = 2
socket_message[2] = function(id, ud , addr)
	local s = socket_pool[id]
	if s == nil then
		return
	end
	-- log remote addr
	if not s.connected then	-- resume may also post connect message
		if s.listen then
			s.addr = addr
			s.port = ud
		end
		s.connected = true
		wakeup(s)
	end
end
```
设置`s.connected = true`，最后`wakeup`将协程放入到wakeup队列，最后再通过`skynet.lua`中的`suspend`唤醒.