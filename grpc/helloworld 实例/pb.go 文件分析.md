##### helloworld.proto 定义

helloworld 例子定义还是比较简单的，一个服务和两个实体


```
service Greeter {
  // 服务动作
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 请求参数
message HelloRequest {
  string name = 1;
}

// 请求相应
message HelloReply {
  string message = 1;
}
```


##### Client API for the service

```
// 客户端接口
type GreeterClient interface {
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

// 接口实现
type greeterClient struct {
	cc *grpc.ClientConn
}

// 接口对象构造, 客户端调用要传入ClientConn指明服务端地址及各种请求配置，如通讯加密等等
func NewGreeterClient(cc *grpc.ClientConn) GreeterClient {
	return &greeterClient{cc}
}

// 接口实例的方法实现
func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := grpc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, c.cc, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```


##### Server API for the service


```
// 服务端接口定义
type GreeterServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
}

//grpc 服务注册
func RegisterGreeterServer(s *grpc.Server, srv GreeterServer) {
	s.RegisterService(&_Greeter_serviceDesc, srv)
}

// 下面一个变量一个函数，主要作用于服务注册
var _Greeter_serviceDesc = grpc.ServiceDesc{
	ServiceName: "helloworld.Greeter",
	HandlerType: (*GreeterServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "SayHello",
			Handler:    _Greeter_SayHello_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "helloworld.proto",
}

func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(HelloRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(GreeterServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/helloworld.Greeter/SayHello",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
	}
	return interceptor(ctx, in, info, handler)
}

```

##### client 和 server 区别


```
grpc 本来就是解决远程调用，让客户端如同调用本地方法一样方便。 如此，grpc 对套接层做了很好的分装，如通讯等等。
所以，client api不仅定义了接口，还提供了接口实现，客户端直接调用方法即可；
而服务端没有提供接口实现，但提供了“服务注册”功能

从而让客户端专注于调用，让服务端专注于业务实现（最后注册就行）
```