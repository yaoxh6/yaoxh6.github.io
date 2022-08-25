---
layout:     post
title:      "自定义rpc学习"
subtitle:   ""
date:       2022-08-25 22:29:00
author:     "Yaoxh6"
catalog: true
header-style: text
tags:
  - lua
  - c++
  - go
  - rpc
---

# demo介绍

写了一个简单的lua到go服务的rpc[demo](https://github.com/yaoxh6/CustomRPC)

只专注实现流程,功能和细节全部不完善,目前只能实现lua侧call到go侧并且得到结果回调处理。

目前项目的不足
- 单对单服务, 缺少名字服务组件,直接用简单的tcp连接,这样在调用远程方法的时候不需要指定服务名, 因为只是简单的实现, 没有服务注册。所以transport组件还有待更新。
- 目前lua_server与其说是server,不如说是client,goserver相对来说更像是server.没有赋予goserver向lua服务call的功能,自动生成代码只生成了goserver作为server的一部分,所以go没法主动call。
- 项目暂时对没有callback的情况未处理,需要在pb文件上做自定义flag,生成代码会复杂很多
- codec用的是json库,对复杂table或者struct没有处理
- 回调函数的位置默认是第二个参数,为了方便处理
# 实现效果

在lua侧新建一个lua_server,通过`lua_server.Send`,向goserver发起rpc调用,`lua_server.Send("SayHello", "hello_reply", "param1")`的含义为调用goserver的`SayHello`方法,参数是`param1`,并且调用结束后将结果返回,接着调用`s2s.hello_reply`,并将go测返回的结果作为参数。

lua侧
``` lua
local SimpleServer = require "SimpleServer"
local lua_server = SimpleServer.create()
function s2s.hello_reply_reply(...)
    log_tree("[hello_reply_reply] param:", {...})
end

function s2s.hello_reply(...)
    log_tree("[hello_reply] param:", {...})
    lua_server.Send("SayHello2", "hello_reply_reply", "param2", 12345)
end
-- 调用go服务的SayHello函数, 参数是param1
-- 回调函数是hello_reply, 即go服务回包是s2s.hello_reply的参数
lua_server.Send("SayHello", "hello_reply", "param1")
```
go侧
```go
func main() {
	h, err := InitServerEnv()
	if err != nil {
		log.Fatal(err)
		return
	}
	pb.RegisterGreeterServer(h, &s2s{})
	if err := rpc.Serve(h); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

type s2s struct{}

func (s *s2s) SayHello(ctx context.Context, helloRequest *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Info(helloRequest)
	return &pb.HelloReply{Message: "HelloReplyContent"}, nil
}

func (s *s2s) SayHello2(ctx context.Context, helloRequest *pb.HelloRequest2) (*pb.HelloReply2, error) {
	log.Info(helloRequest)
	return &pb.HelloReply2{ReplyNum:helloRequest.Num, Res: true}, nil
}
```
pb文件
```proto
syntax = "proto3";

option go_package = "./;pb";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  rpc SayHello2 (HelloRequest2) returns (HelloReply2) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

message HelloRequest2 {
  string request_name = 1;
  int32 num = 2;
}

message HelloReply2 {
  int32 reply_num = 1;
  bool res = 2;
}
```

输出结果:

首先lua调用go侧的`SayHello`方法,得到go侧的回包`HelloReplyContent`,然后执行callback函数`hello_reply`,在函数内部再次调用go侧的`SayHello2`方法,将传来的数字和true返回,最后执行lua侧的`hello_reply_reply`函数,将`12345, true`打印出来
![res](/img/in-post/post-customrpc/res.png)

# demo运行
- go服务启动, 先在根目录`make`或者`make custom`,该命令会生成`*.pb.go`和`*_rpc.custom.go`文件
进入examples/helloworld/go_server/目录,`go run .`

- lua服务启动,进入lua-server目录执行build.bat,进入build,双击lua-server.sln,打开项目直接F5启动

# 思路

## lua与c++
将lua与go通信转换成c++与go通信。利用[luna](https://github.com/trumanzhao/luna)将c++类`SimpleServer`导出给lua使用.c++使用简单的tcp连接,codec使用[json库](https://github.com/nlohmann/json).

通过lua传入的参数依次识别转化到json
```c++
bool EncodeData(lua_State* L, json_t& j, int index) {
	int type = lua_type(L, index);
	switch (type) {
	case LUA_TNIL:
		j[index-1] = "";
		return true;
	case LUA_TNUMBER:
		lua_isinteger(L, index) ? j[index-1] = (lua_tointeger(L, index)) : j[index-1] = (lua_tonumber(L, index));
		return true;
	case LUA_TBOOLEAN:
		j[index-1] = (!!lua_toboolean(L, index));
		return true;
	case LUA_TSTRING:
		j[index-1] = lua_tostring(L, index);
		return true;
	case LUA_TTABLE:
		//暂时不支持table
		return false;
	default:
		break;
	}
	return false;
}

int SimpleServer::Send(lua_State* L) {
    ...
	int top = lua_gettop(L);
	for (int i = 1; i<=top; i++)
	{
		if (!EncodeData(L, j, i)) {
			printf("EncodeData Err index = %d\n", i);
			return 0;
		}
	}
    ...
}
```
获取回包是相反的操作
```c++
bool DecodeData(lua_State* L, json_t& j, int index) {
	auto type = j[index].type();
	switch (type)
	{
	case json_t::value_t::null:
		lua_pushnil(L);
		return true;
	case json_t::value_t::number_integer:
		lua_pushinteger(L, j[index]);
		return true;
	case json_t::value_t::number_unsigned:
	case json_t::value_t::number_float:
		lua_pushnumber(L, j[index]);
		return true;
	case json_t::value_t::boolean:
		lua_pushboolean(L, j[index]);
		return true;
	case json_t::value_t::string:
		lua_pushstring(L, j[index].get<std::string>().c_str());
		return true;
	default:
		break;
	}
	return false;
}

void SimpleServer::Recv(lua_State* L, const char* data, size_t data_len) {
    ...
    json_t j;
	try {
		j = json_t::parse(data);
	}
	catch (std::exception& e)
	{
		std::cout << e.what() << std::endl;
	}
	string callback = j[0];
	lua_pushstring(L, callback.c_str());
	for (int i = 1; i < j.size(); i++) {
		if (!DecodeData(L, j, i)) {
			printf("DecodeData Err index = %d\n", i);
			return;
		}
	}
    ...
}
```
### c++到lua的回调实现

lua侧统一接口`on_call_with_handle`,`msg`是回调函数名,函数名统一注册在`s2s`中,所以直接取出对应的处理函数.
```lua
lua_server.on_call_with_handle = function(msg, ...)
    log_tree("msg", {msg, ...})
    local proc = s2s[msg]
    if not proc then
        print("function ", msg, " is not exist")
        return
    end
    local ok, err = xpcall(proc, debug.traceback, ...)
    if not ok then
        print("function ", msg, " execution failed err", err)
        return
    end
end
```

c++侧是luna中的辅助函数`lua_get_object_function`,具体的作用就是在lua_server这个table中`lua_getfield(L, -1, function)`,取出的函数自动放在栈顶, 然后json解包之后,指定参数,再`lua_call_function`即可
```c++
	if (!lua_get_object_function(L, this, "on_call_with_handle")) {
		printf("SimpleServer::Recv on_call_with_hanldle failed\n");
		return;
	}
    ...
    lua_call_function(L, nullptr, j.size(), 0);
```
## go服务
go重写了`transport`,`codec`,`service`

### transport
Package定义
```go
type Package struct {
	ServiceName string
	Data []byte
}
```
Transport接口, 只要满足下面接口就可以作为transport,目前tranport的实现,`Recv`是从conn中read, `Send`是向conn中write,较为简单的同步实现逻辑。
```go
type Transport interface {
	Setup(network string, address string) error
	Recv() (*Package, error)
	Send(pak *Package) error
}
```
### codec
codec就是编码与解码,这也不做太多的自定义,直接使用go自带的json库, 简单按照接口封一层

codec接口
```go
type Codec interface {
	Encode(interface{}) ([]byte, error)
	Decode([]byte, interface{}) error
}
```
### service
- `Register`实现处理函数的注册,即为`serviceHandler`赋值,这样可以通过`HandleRPC`调用具体的业务处理
- `Serve`主循环, 从`transport`中取出数据包,每次通信开一个go线程处理.

```go
type Service interface {
	Register(serviceDesc interface{}, serviceImpl interface{}) error
	Serve() error
	Close(chan struct{}) error
}

type ServiceHandler interface {
	Name() string
	HandleRPC(context.Context, string, Codec, *transport.Package) ([]byte, error)
}

type CustomService struct {
	requestId      int64
	ctx            context.Context
	cancel         context.CancelFunc
	trans          transport.Transport
	serviceHandler ServiceHandler
	d              Codec
}
```

具体流程, 从transport中取出pak,go线程处理,首先Decode,零号位置是rpcName,剩下的是回调函数(如果有)和参数,通过handleRPC处理,处理之后如果有回包就send回去, send也是transport实现。

```go
func (h *CustomService) internalRecv() (int, error) {
    pak, err := h.trans.Recv()
    ...
    go func(pak *transport.Package) {
        ...
        h.internalHandle(pak)
    }(pak)
}

func (h *CustomService) internalHandle(pak *transport.Package) {
	var param []interface{}
	var rpcName string
	err := h.d.Decode(pak.Data, &param)
	rpcName = param[0].(string)
    ...
	rspBin, err := h.handleRPC(rpcName, pak)
	if err == nil && len(rspBin) > 0 {
		var sendPak = *pak
		sendPak.Data = rspBin
		err = h.Send(&sendPak)
	}
}

func (h *CustomService) Send(pak *transport.Package) error {
	return h.trans.Send(pak)
}
```

handleRPC调用具体业务代码, 知道了rpcName,就直接通过反射得到对应的函数, 每个函数自动生成
```go
func (h *greeterHandler) HandleRPC(ctx context.Context, rpcName string, d rpc.Codec, pak *transport.Package) ([]byte, error) {
	value := reflect.ValueOf(h)
	args := []reflect.Value{reflect.ValueOf(ctx), reflect.ValueOf(rpcName), reflect.ValueOf(d), reflect.ValueOf(pak)}
	f := value.MethodByName(rpcName)
	res := f.Call(args)
	if res[0].IsNil() {
		return nil, res[1].Interface().(error)
	}
	return res[0].Bytes(), nil
}
```
以`SayHello`为例, 思路是先得到request和reply对应的pb,这样可以遍历得到对应参数的具体字段名称,然后通过codec解出每个字段的具体值,再调用业务具体的函数,得到返回值,返回值再编码完成框架中调用业务代码的整个处理。
```go
func (h *greeterHandler) SayHello(ctx context.Context, rpcName string, d rpc.Codec, pak *transport.Package) ([]byte, error) {
	var helloRequest HelloRequest
	var inVarList []interface{}
	var temp string
	var callback string
	inVarList = append(inVarList, &temp)
	inVarList = append(inVarList, &callback)
	inVarList = append(inVarList, &helloRequest.Name)
	err := rpc.DecodeArchiverWithTrace(rpcName, d, pak, inVarList...)
	if err != nil {
		return nil, err
	}
	helloReply, err := h.GreeterCustomServer.SayHello(ctx, &helloRequest)
	if err != nil {
		return nil, fmt.Errorf(`rpc failed in [%s]: %s`, rpcName, err.Error())
	}
	var outVarList []interface{}
	outVarList = append(outVarList, &callback)
	outVarList = append(outVarList, &helloReply.Message)
	return d.Encode(outVarList)
}

func (g *custom) generateServerMethod(servName string, method *pb.MethodDescriptorProto, messages []*descriptorpb.DescriptorProto) string {
	methName := generator.CamelCase(method.GetName())
	hname := methName
	serveType := servName + "Handler"
	inType := g.typeName(method.GetInputType())
	outType := g.typeName(method.GetOutputType())
	inputPb := g.getPbByName(messages, inType)
	outPb := g.getPbByName(messages, outType)
	if !method.GetClientStreaming() && !method.GetServerStreaming() {
		g.P("func (h *", unexport(servName), "Handler) ", hname, "(ctx ", contextPkg, ".Context, rpcName string, d rpc.Codec, pak *", transportPkg, ".Package) ([]byte, error) {")
		g.P("var ", unexport(inType), " ", inType)
		g.P("var inVarList []interface{}")
		g.P("var temp string")
		g.P("var callback string")
		g.P("inVarList = append(inVarList, &temp)")
		g.P("inVarList = append(inVarList, &callback)")
		// ...
		for _, field := range inputPb.GetField() {
			g.P("inVarList = append(inVarList, &", unexport(inType), ".", generator.CamelCase(field.GetName()), ")")
		}
		g.P("err := rpc.DecodeArchiverWithTrace(rpcName, d, pak, inVarList...)")
		g.P("if err != nil {")
		g.P("return nil, err")
		g.P("}")
		g.P(unexport(outType), ", err := h.", servName, "CustomServer.", methName, "(ctx, &", unexport(inType),")")
		g.P("if err != nil {")
		g.P("return nil, fmt.Errorf(`rpc failed in [%s]: %s`, rpcName, err.Error())")
		g.P("}")
		g.P("var outVarList []interface{}")
		g.P("outVarList = append(outVarList, &callback)")
		// ...
		for _, field := range outPb.GetField() {
			g.P("outVarList = append(outVarList, &", unexport(outType), ".", generator.CamelCase(field.GetName()), ")")
		}
		g.P("return d.Encode(outVarList)")
		g.P("}")
		g.P()
		return hname
	}
    ...
}
```