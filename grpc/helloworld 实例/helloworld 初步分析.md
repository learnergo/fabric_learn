##### helloworld.proto ����

helloworld ���Ӷ��廹�ǱȽϼ򵥵ģ�һ�����������ʵ��


```
service Greeter {
  // ������
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// �������
message HelloRequest {
  string name = 1;
}

// ������Ӧ
message HelloReply {
  string message = 1;
}
```


##### Client API for the service

```
// �ͻ��˽ӿ�
type GreeterClient interface {
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

// �ӿ�ʵ��
type greeterClient struct {
	cc *grpc.ClientConn
}

// �ӿڶ�����
func NewGreeterClient(cc *grpc.ClientConn) GreeterClient {
	return &greeterClient{cc}
}

// �ӿ�ʵ���ķ���ʵ��
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
// ����˽ӿڶ���
type GreeterServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
}

//grpc ����ע��
func RegisterGreeterServer(s *grpc.Server, srv GreeterServer) {
	s.RegisterService(&_Greeter_serviceDesc, srv)
}

// ����һ������һ����������Ҫ�����ڷ���ע��
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

##### client �� server ����


```
grpc �������ǽ��Զ�̵��ã��ÿͻ�����ͬ���ñ��ط���һ�����㡣 ��ˣ�grpc ���׽Ӳ����˺ܺõķ�װ����ͨѶ�ȵȡ�
���ԣ�client api���������˽ӿڣ����ṩ�˽ӿ�ʵ�֣��ͻ���ֱ�ӵ��÷������ɣ�
�������û���ṩ�ӿ�ʵ�֣����ṩ�ˡ�����ע�ᡱ����

�Ӷ��ÿͻ���רע�ڵ��ã��÷����רע��ҵ��ʵ�֣����ע����У�
```